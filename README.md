#region Using declarations
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.ComponentModel.DataAnnotations;
using System.Globalization;
using System.Xml.Serialization;
using System.Windows.Media;
using NinjaTrader.Data;
using NinjaTrader.Gui.Tools;
using NinjaTrader.NinjaScript;
using NinjaTrader.NinjaScript.DrawingTools;
#endregion

namespace NinjaTrader.NinjaScript.Indicators
{
	// ---------------------------
	// Enums (match TV options)
	// ---------------------------
	public enum SRInvalidationMode { Wick, Close }
	public enum SRInvalidationModel { Wick = SRInvalidationMode.Wick, Close = SRInvalidationMode.Close } // compatibility alias
	public enum ExpandLinesMode { None, OnlyValid, All }
	public enum ExpandLinesModel { None = ExpandLinesMode.None, OnlyValid = ExpandLinesMode.OnlyValid, All = ExpandLinesMode.All } // compatibility alias
	public enum SRStyleMode { Lines, Zones }
	public enum SRLineStyleMode { Solid, Dash, Dot }
	public enum TimeframeUnit { Minute, Hour, Day, Week }

	public class ZLEMASRHTF : Indicator
	{
		private enum SRType { Support, Resistance }

		private const int    ATR_LEN                = 20;
		private const int    MAX_SR_PER_SERIES      = 10;
		private const int    MIN_SR_BARS            = 5;
		private const double TOO_CLOSE_ATR_FRAC     = 1.0 / 8.0; // matches TV
		private const int    RETEST_COOLDOWN_BARS   = 3;
		private const int    MAX_RETEST_MARKERS     = 50;
		private const int    PRIMARY_TIME_LOOKBACK = 5000; // for time->barsAgo mapping
		private const double ZONE_ATR_MULT          = 0.075; // matches TV (zoneSize * 0.075 * ATR)

		// ---------------------------
		// Data structures
		// ---------------------------
		private class SRInfo
		{
			public DateTime StartTime;
			public int      StartBarIndex; // in its own series
			public double   Price;
			public SRType   Type;
			public int      Strength;
			public string   TimeframeLabel;
			public bool     Ephemeral;

			public DateTime? BreakTime;
			public int?      BreakBarIndex; // in its own series

			// alert gating
			public bool BreakAlertFired;
		}

		private class SROverlay
		{
			public string Key;
			public SRInfo Info;
			public string CombinedLabel;

			// retests are calculated on PRIMARY series (like TV does for display)
			public List<DateTime> RetestTimes = new List<DateTime>();
			public int LastRetestPrimaryBarIndex = int.MinValue;
		}

		private struct BarsPeriodKey : IEquatable<BarsPeriodKey>
		{
			public BarsPeriodType Type;
			public int Value;

			public BarsPeriodKey(BarsPeriodType type, int value)
			{
				Type = type;
				Value = value;
			}

			public bool Equals(BarsPeriodKey other) => Type == other.Type && Value == other.Value;
			public override bool Equals(object obj) => obj is BarsPeriodKey other && Equals(other);
			public override int GetHashCode() => ((int)Type * 397) ^ Value;
		}

		// ---------------------------
		// Series / state
		// ---------------------------
		private List<SRInfo>[] srByBip;
		private ATR[] atrByBip;
		private SMA[] volSmaByBip;

		private readonly Dictionary<string, SROverlay> overlaysByKey = new Dictionary<string, SROverlay>();
		private HashSet<string> lastDrawTags = new HashSet<string>();

		// which BarsInProgress indices feed levels
		private readonly HashSet<int> levelSourceBips = new HashSet<int>();
		private readonly Dictionary<int, string> bipLabel = new Dictionary<int, string>();

		private int tf1Bip = -1;
		private int tf2Bip = -1;
		private int tf3Bip = -1;

		private readonly Dictionary<BarsPeriodKey, int> addedBips = new Dictionary<BarsPeriodKey, int>();

		private const string TAG_PREFIX = "ZLEMASRHTF";

		// ---------------------------
		// NinjaScript lifecycle
		// ---------------------------
		protected override void OnStateChange()
		{
			if (State == State.SetDefaults)
			{
				Name					= "ZLEMASRHTF";
				Description				= "MTF Pivot-based Support & Resistance with Breaks/Retests (NT8 conversion).";
				Calculate				= Calculate.OnBarClose;
				IsOverlay				= true;
				DisplayInDataBox		= false;
				DrawOnPricePanel		= true;
				PaintPriceMarkers		= true;
				IsSuspendedWhileInactive = true;

				// --- Defaults to match your screenshots ---
				PivotLength			= 15;
				Strength			= 1;
				Invalidation		= SRInvalidationMode.Close;
				ExpandLines			= ExpandLinesMode.OnlyValid;
				ShowInvalidated		= false;

				Timeframe1Enabled	= true;
				Timeframe1Unit		= TimeframeUnit.Hour;
				Timeframe1Value		= 4;

				Timeframe2Enabled	= true;
				Timeframe2Unit		= TimeframeUnit.Minute;
				Timeframe2Value		= 15;

				Timeframe3Enabled	= false;
				Timeframe3Unit		= TimeframeUnit.Week;
				Timeframe3Value		= 1;

				ShowBreaks			= true;
				ShowRetests			= true;
				AvoidFalseBreaks	= true;
				BreakVolumeThreshold = 0.3;
				InverseColorAfterBroken = false;

				StyleMode			= SRStyleMode.Lines;
				LineStyle			= SRLineStyleMode.Solid;
				LineWidth			= 5;
				ZoneWidth			= 1.0;

				SupportBrush		= Brushes.Blue;
				ResistanceBrush		= Brushes.Red;
				BreakBrush			= Brushes.Blue;
				TextBrush			= Brushes.Gray;

				EnableRetestAlerts	= false;
				EnableBreakAlerts	= false;

				ShowLines			= true;
				ShowBoxes			= false;
				ShowPaneLabels		= false;
			}
			else if (State == State.Configure)
			{
				// Configure MTF data series
				levelSourceBips.Clear();
				bipLabel.Clear();
				addedBips.Clear();

				int nextBip = 1;
				tf1Bip = ConfigureTimeframe(Timeframe1Enabled, Timeframe1Unit, Timeframe1Value, FormatTimeframeLabel(Timeframe1Unit, Timeframe1Value), ref nextBip);
				tf2Bip = ConfigureTimeframe(Timeframe2Enabled, Timeframe2Unit, Timeframe2Value, FormatTimeframeLabel(Timeframe2Unit, Timeframe2Value), ref nextBip);
				tf3Bip = ConfigureTimeframe(Timeframe3Enabled, Timeframe3Unit, Timeframe3Value, FormatTimeframeLabel(Timeframe3Unit, Timeframe3Value), ref nextBip);
			}
			else if (State == State.DataLoaded)
			{
				srByBip   = new List<SRInfo>[BarsArray.Length];
				atrByBip  = new ATR[BarsArray.Length];
				volSmaByBip = new SMA[BarsArray.Length];

				for (int i = 0; i < BarsArray.Length; i++)
				{
					srByBip[i] = new List<SRInfo>();
					atrByBip[i] = ATR(Closes[i], ATR_LEN);
					volSmaByBip[i] = SMA(Volumes[i], ATR_LEN);

					// fallback label if not assigned (e.g., primary)
					if (!bipLabel.ContainsKey(i))
						bipLabel[i] = FormatBarsPeriodLabel(BarsArray[i].BarsPeriod);
				}
			}
		}

		protected override void OnBarUpdate()
		{
			// Always need primary present for overlay rendering
			if (CurrentBars[0] < Math.Max(ATR_LEN, 2 * PivotLength + 2))
				return;

			// Update S/R lists on whichever series are sources
			if (ShouldProcessLevelsForBip(BarsInProgress))
			{
				if (CurrentBars[BarsInProgress] >= Math.Max(ATR_LEN, 2 * PivotLength + 2))
					ProcessLevelSourceSeries(BarsInProgress);
			}

			// Render + retests are done on PRIMARY (BIP 0)
			if (BarsInProgress == 0)
			{
				var overlays = BuildOverlaysToRender();
				UpdatePrimaryRetests(overlays);
				RenderOverlays(overlays);
			}
		}

		// ---------------------------
		// Core logic: SR detection per timeframe series
		// ---------------------------
		private void ProcessLevelSourceSeries(int bip)
		{
			// 1) Breaks + strength updates
			UpdateBreaksAndStrength(bip);

			// 2) New pivots
			DetectNewPivots(bip);

			// 3) Trim list size
			if (srByBip[bip].Count > MAX_SR_PER_SERIES)
				srByBip[bip].RemoveRange(MAX_SR_PER_SERIES, srByBip[bip].Count - MAX_SR_PER_SERIES);
		}

		private void UpdateBreaksAndStrength(int bip)
		{
			double invHigh = (Invalidation == SRInvalidationMode.Close) ? Closes[bip][0] : Highs[bip][0];
			double invLow  = (Invalidation == SRInvalidationMode.Close) ? Closes[bip][0] : Lows[bip][0];

			double vol   = Volumes[bip][0];
			double avgVol = volSmaByBip[bip][0];

			DateTime barTime = Times[bip][0];
			int barIndex = CurrentBars[bip];

			List<SRInfo> toAddEphemeral = null;

			for (int i = 0; i < srByBip[bip].Count; i++)
			{
				SRInfo sr = srByBip[bip][i];

				// Break detection
				if (!sr.BreakTime.HasValue)
				{
					bool broke =
						(sr.Type == SRType.Resistance && invHigh > sr.Price) ||
						(sr.Type == SRType.Support    && invLow  < sr.Price);

					if (broke)
					{
						bool passVol = !AvoidFalseBreaks || (avgVol > 0 && vol > avgVol * BreakVolumeThreshold);

						if (passVol)
						{
							sr.BreakTime = barTime;
							sr.BreakBarIndex = barIndex;

							// Alerts (optional)
							if (EnableBreakAlerts && !sr.BreakAlertFired)
							{
								sr.BreakAlertFired = true;
								Alert($"{TAG_PREFIX}_Break_{bip}_{sr.Price.ToString("0.########", CultureInfo.InvariantCulture)}",
									Priority.Medium,
									$"Break: {sr.Type} @ {FormatPrice(sr.Price)} ({sr.TimeframeLabel})",
									NinjaTrader.Core.Globals.InstallDir + @"\sounds\Alert1.wav",
									0,
									BreakBrush,
									TextBrush);
							}

							// Inverse after broken (optional)
							if (InverseColorAfterBroken && !sr.Ephemeral && sr.Strength >= Strength)
							{
								if (toAddEphemeral == null)
									toAddEphemeral = new List<SRInfo>();

								toAddEphemeral.Add(new SRInfo
								{
									StartTime = barTime,
									StartBarIndex = barIndex,
									Price = sr.Price,
									Type = (sr.Type == SRType.Resistance) ? SRType.Support : SRType.Resistance,
									Strength = sr.Strength,
									TimeframeLabel = sr.TimeframeLabel,
									Ephemeral = true,
									BreakTime = null,
									BreakBarIndex = null,
									BreakAlertFired = false
								});
							}
						}
					}
				}

				// Strength updates (retests within this timeframe series)
				if (!sr.BreakTime.HasValue && barTime > sr.StartTime)
				{
					bool isRetest =
						(sr.Type == SRType.Resistance && Highs[bip][0] >= sr.Price && Closes[bip][0] <= sr.Price) ||
						(sr.Type == SRType.Support    && Lows[bip][0]  <= sr.Price && Closes[bip][0] >= sr.Price);

					if (isRetest && !sr.Ephemeral)
						sr.Strength += 1;
				}
			}

			// Add ephemeral at front (most recent)
			if (toAddEphemeral != null && toAddEphemeral.Count > 0)
			{
				for (int k = toAddEphemeral.Count - 1; k >= 0; k--)
					srByBip[bip].Insert(0, toAddEphemeral[k]);
			}
		}

		private void DetectNewPivots(int bip)
		{
			int L = PivotLength;
			if (CurrentBars[bip] < 2 * L + 1)
				return;

			double candHigh = Highs[bip][L];
			double candLow  = Lows[bip][L];

			bool isPivotHigh = true;
			bool isPivotLow  = true;

			for (int i = 0; i <= 2 * L; i++)
			{
				if (i == L) continue;

				if (Highs[bip][i] >= candHigh)
					isPivotHigh = false;
				if (Lows[bip][i] <= candLow)
					isPivotLow = false;

				if (!isPivotHigh && !isPivotLow)
					break;
			}

			double atr = atrByBip[bip][0];
			double tooClose = atr * TOO_CLOSE_ATR_FRAC;

			if (isPivotLow)
			{
				double price = RoundToTick(candLow);
				if (IsNewLevelFarEnough(bip, price, tooClose))
				{
					srByBip[bip].Insert(0, new SRInfo
					{
						StartTime = Times[bip][L],
						StartBarIndex = CurrentBars[bip] - L,
						Price = price,
						Type = SRType.Support,
						Strength = 1,
						TimeframeLabel = bipLabel.ContainsKey(bip) ? bipLabel[bip] : "TF",
						Ephemeral = false
					});
				}
			}

			if (isPivotHigh)
			{
				double price = RoundToTick(candHigh);
				if (IsNewLevelFarEnough(bip, price, tooClose))
				{
					srByBip[bip].Insert(0, new SRInfo
					{
						StartTime = Times[bip][L],
						StartBarIndex = CurrentBars[bip] - L,
						Price = price,
						Type = SRType.Resistance,
						Strength = 1,
						TimeframeLabel = bipLabel.ContainsKey(bip) ? bipLabel[bip] : "TF",
						Ephemeral = false
					});
				}
			}
		}

		private bool IsNewLevelFarEnough(int bip, double price, double tooClose)
		{
			for (int i = 0; i < srByBip[bip].Count; i++)
			{
				var sr = srByBip[bip][i];
				if (!sr.BreakTime.HasValue && Math.Abs(sr.Price - price) < tooClose)
					return false;
			}
			return true;
		}

		// ---------------------------
		// Build overlays + merge close levels (like TV)
		// ---------------------------
		private List<SROverlay> BuildOverlaysToRender()
		{
			var candidates = new List<SRInfo>();
			var seenBips = new HashSet<int>();

			// Pull from enabled timeframes (tf1/2/3), de-duping the bips
			if (tf1Bip >= 0) seenBips.Add(tf1Bip);
			if (tf2Bip >= 0) seenBips.Add(tf2Bip);
			if (tf3Bip >= 0) seenBips.Add(tf3Bip);

			foreach (int bip in seenBips)
			{
				if (bip < 0 || bip >= BarsArray.Length)
					continue;

				for (int i = 0; i < srByBip[bip].Count; i++)
				{
					var sr = srByBip[bip][i];

					if (sr.Strength < Strength)
						continue;

					if (sr.BreakBarIndex.HasValue && (sr.BreakBarIndex.Value - sr.StartBarIndex) < MIN_SR_BARS)
						continue;

					if (sr.BreakTime.HasValue && !ShowInvalidated)
						continue;

					candidates.Add(sr);
				}
			}

			// Sort newest-first
			candidates.Sort((a, b) => b.StartTime.CompareTo(a.StartTime));

			double primaryAtr = atrByBip[0][0];
			double mergeThreshold = primaryAtr * TOO_CLOSE_ATR_FRAC;

			var overlays = new List<SROverlay>();

			foreach (var sr in candidates)
			{
				// Merge close levels with same type/ephemeral
				SROverlay existing = null;
				for (int j = 0; j < overlays.Count; j++)
				{
					var o = overlays[j];
					if (o.Info.Type == sr.Type && o.Info.Ephemeral == sr.Ephemeral && Math.Abs(o.Info.Price - sr.Price) <= mergeThreshold)
					{
						existing = o;
						break;
					}
				}

				if (existing != null)
				{
					existing.CombinedLabel = MergeLabel(existing.CombinedLabel, sr.TimeframeLabel);
					continue;
				}

				string key = CreateOverlayKey(sr.Type, sr.Ephemeral, sr.Price);

				if (!overlaysByKey.TryGetValue(key, out SROverlay overlay))
				{
					overlay = new SROverlay { Key = key };
					overlaysByKey[key] = overlay;
				}

				overlay.Info = sr;
				overlay.CombinedLabel = sr.TimeframeLabel;
				overlays.Add(overlay);
			}

			return overlays;
		}

		// ---------------------------
		// Retests on PRIMARY series (for display/alerts)
		// ---------------------------
		private void UpdatePrimaryRetests(List<SROverlay> overlays)
		{
			if (!ShowRetests && !EnableRetestAlerts)
				return;

			DateTime now = Times[0][0];
			int primaryBar = CurrentBars[0];

			for (int i = 0; i < overlays.Count; i++)
			{
				var o = overlays[i];
				var sr = o.Info;

				// Only if not broken
				if (sr.BreakTime.HasValue)
					continue;

				// Must be after start
				if (now <= sr.StartTime)
					continue;

				bool retest =
					(sr.Type == SRType.Resistance && Highs[0][0] >= sr.Price && Closes[0][0] <= sr.Price) ||
					(sr.Type == SRType.Support    && Lows[0][0]  <= sr.Price && Closes[0][0] >= sr.Price);

				if (!retest)
					continue;

				// cooldown (like TV)
				if (primaryBar - o.LastRetestPrimaryBarIndex < RETEST_COOLDOWN_BARS)
					continue;

				o.LastRetestPrimaryBarIndex = primaryBar;
				o.RetestTimes.Insert(0, now);

				if (o.RetestTimes.Count > MAX_RETEST_MARKERS)
					o.RetestTimes.RemoveAt(o.RetestTimes.Count - 1);

				if (EnableRetestAlerts)
				{
					Alert($"{TAG_PREFIX}_Retest_{o.Key}",
						Priority.Low,
						$"Retest: {sr.Type} @ {FormatPrice(sr.Price)} ({o.CombinedLabel})",
						NinjaTrader.Core.Globals.InstallDir + @"\sounds\Alert2.wav",
						0,
						(sr.Type == SRType.Resistance ? ResistanceBrush : SupportBrush),
						TextBrush);
				}
			}
		}

		// ---------------------------
		// Rendering
		// ---------------------------
		private void RenderOverlays(List<SROverlay> overlays)
		{
			var curTags = new HashSet<string>();
			DashStyleHelper dash = GetDashStyle();

			for (int i = 0; i < overlays.Count; i++)
			{
				var o = overlays[i];
				var sr = o.Info;

				Brush levelBrush = (sr.Type == SRType.Resistance) ? ResistanceBrush : SupportBrush;

				// Lines / Zones
				if (StyleMode == SRStyleMode.Lines && ShowLines)
				{
					DrawSRLine(o, sr, levelBrush, dash, curTags);
				}
				else if (StyleMode == SRStyleMode.Zones && ShowBoxes)
				{
					DrawSRZone(o, sr, levelBrush, curTags);
				}

				// Pane labels (off by default to match your screenshot)
				if (ShowPaneLabels)
				{
					string lblTag = $"{TAG_PREFIX}_LBL_{o.Key}";
					string txt = $"{o.CombinedLabel} | {FormatPrice(sr.Price)}";
					Draw.Text(this, lblTag, txt, 0, sr.Price, TextBrush);
					curTags.Add(lblTag);
				}

				// Break marker (B)
				if (ShowBreaks && sr.BreakTime.HasValue)
				{
					int barsAgo = GetPrimaryBarsAgo(sr.BreakTime.Value);
					if (barsAgo >= 0 && barsAgo <= CurrentBars[0])
					{
						double y = (sr.Type == SRType.Resistance)
							? (Lows[0][barsAgo] - 2 * TickSize)
							: (Highs[0][barsAgo] + 2 * TickSize);

						string bTag = $"{TAG_PREFIX}_BRK_{o.Key}_{sr.BreakTime.Value.Ticks}";
						Draw.Text(this, bTag, "B", barsAgo, y, BreakBrush);
						curTags.Add(bTag);
					}
				}

				// Retest markers (R)
				if (ShowRetests && o.RetestTimes.Count > 0)
				{
					int drawn = 0;
					foreach (var rt in o.RetestTimes)
					{
						if (drawn++ >= MAX_RETEST_MARKERS)
							break;

						// don't plot retests outside SR lifetime
						if (rt < sr.StartTime)
							continue;
						if (sr.BreakTime.HasValue && rt >= sr.BreakTime.Value)
							continue;

						int barsAgo = GetPrimaryBarsAgo(rt);
						if (barsAgo < 0 || barsAgo > CurrentBars[0])
							continue;

						double y = (sr.Type == SRType.Resistance)
							? (Highs[0][barsAgo] + 2 * TickSize)
							: (Lows[0][barsAgo] - 2 * TickSize);

						string rTag = $"{TAG_PREFIX}_RT_{o.Key}_{rt.Ticks}";
						Draw.Text(this, rTag, "R", barsAgo, y, levelBrush);
						curTags.Add(rTag);
					}
				}
			}

			// Remove old draw objects that are no longer needed
			foreach (var oldTag in lastDrawTags)
			{
				if (!curTags.Contains(oldTag))
					RemoveDrawObject(oldTag);
			}
			lastDrawTags = curTags;
		}

		private void DrawSRLine(SROverlay o, SRInfo sr, Brush brush, DashStyleHelper dash, HashSet<string> tags)
		{
			string lineTag = $"{TAG_PREFIX}_LN_{o.Key}";

			bool isBroken = sr.BreakTime.HasValue;

			// Match TV behavior:
			// - OnlyValid: active lines extend both; broken lines are segment (if shown)
			// - All: extend both regardless of break
			// - None: active extend right; broken segment
			if (!isBroken)
			{
				if (ExpandLines == ExpandLinesMode.OnlyValid || ExpandLines == ExpandLinesMode.All)
				{
					var h = Draw.HorizontalLine(this, lineTag, sr.Price, brush);
					if (h != null)
					{
						h.Stroke.Width = LineWidth;
						h.Stroke.DashStyleHelper = dash;
					}
					tags.Add(lineTag);
					return;
				}
				else
				{
					int startBarsAgo = GetPrimaryBarsAgo(sr.StartTime);
					if (startBarsAgo < 0) startBarsAgo = Math.Min(CurrentBars[0], 200);

					var ray = Draw.Ray(this, lineTag, false, startBarsAgo, sr.Price, 0, sr.Price, brush, dash, LineWidth);
					tags.Add(lineTag);
					return;
				}
			}
			else
			{
				if (ExpandLines == ExpandLinesMode.All)
				{
					var h = Draw.HorizontalLine(this, lineTag, sr.Price, brush);
					if (h != null)
					{
						h.Stroke.Width = LineWidth;
						h.Stroke.DashStyleHelper = dash;
					}
					tags.Add(lineTag);
					return;
				}

				// segment start -> break
				int sAgo = GetPrimaryBarsAgo(sr.StartTime);
				int bAgo = GetPrimaryBarsAgo(sr.BreakTime.Value);

				if (sAgo < 0 || bAgo < 0)
				{
					// fallback
					var h = Draw.HorizontalLine(this, lineTag, sr.Price, brush);
					if (h != null)
					{
						h.Stroke.Width = LineWidth;
						h.Stroke.DashStyleHelper = dash;
					}
					tags.Add(lineTag);
					return;
				}

				var ln = Draw.Line(this, lineTag, false, sAgo, sr.Price, bAgo, sr.Price, brush, dash, LineWidth);
				tags.Add(lineTag);
				return;
			}
		}

		private void DrawSRZone(SROverlay o, SRInfo sr, Brush brush, HashSet<string> tags)
		{
			// Zones are optional; your screenshot uses Lines, but this keeps parity with the TV script.
			double atr = atrByBip[0][0];
			double half = atr * (ZoneWidth * ZONE_ATR_MULT);

			double top = sr.Price + half;
			double bot = sr.Price - half;

			string zTag = $"{TAG_PREFIX}_ZN_{o.Key}";

			int leftBarsAgo = Math.Min(CurrentBars[0], 200);
			int rightBarsAgo = 0;

			// Make it span the visible right side; for "OnlyValid"/"All" we'd like wide zones.
			// (NT rectangles do not "extend", so we approximate with a large lookback.)
			Draw.Rectangle(this, zTag, false, leftBarsAgo, top, rightBarsAgo, bot, null, brush, 15);
			tags.Add(zTag);
		}

		// ---------------------------
		// Helpers
		// ---------------------------
		private bool ShouldProcessLevelsForBip(int bip) => levelSourceBips.Contains(bip);

		private int ConfigureTimeframe(bool enabled, TimeframeUnit unit, int value, string label, ref int nextBip)
		{
			if (!enabled || value <= 0)
				return -1;

			ConvertTimeframe(unit, value, out BarsPeriodType bpt, out int bpv);

			// If requested TF matches primary, use BIP 0
			if (BarsPeriod.BarsPeriodType == bpt && BarsPeriod.Value == bpv)
			{
				levelSourceBips.Add(0);
				bipLabel[0] = label;
				return 0;
			}

			// De-dupe identical TFs
			var key = new BarsPeriodKey(bpt, bpv);
			if (addedBips.TryGetValue(key, out int existingBip))
			{
				levelSourceBips.Add(existingBip);
				bipLabel[existingBip] = label;
				return existingBip;
			}

			AddDataSeries(bpt, bpv);
			int bip = nextBip;
			nextBip++;

			addedBips[key] = bip;
			levelSourceBips.Add(bip);
			bipLabel[bip] = label;

			return bip;
		}

		private void ConvertTimeframe(TimeframeUnit unit, int value, out BarsPeriodType type, out int val)
		{
			switch (unit)
			{
				case TimeframeUnit.Minute:
					type = BarsPeriodType.Minute;
					val = value;
					return;
				case TimeframeUnit.Hour:
					type = BarsPeriodType.Minute;
					val = Math.Max(1, value) * 60;
					return;
				case TimeframeUnit.Day:
					type = BarsPeriodType.Day;
					val = value;
					return;
				case TimeframeUnit.Week:
					type = BarsPeriodType.Week;
					val = value;
					return;
				default:
					type = BarsPeriodType.Minute;
					val = value;
					return;
			}
		}

		private string FormatTimeframeLabel(TimeframeUnit unit, int value)
		{
			switch (unit)
			{
				case TimeframeUnit.Hour:
					return value == 1 ? "1 Hour" : $"{value} Hours";
				case TimeframeUnit.Day:
					return value == 1 ? "1 Day" : $"{value} Days";
				case TimeframeUnit.Week:
					return value == 1 ? "1 Week" : $"{value} Weeks";
				default:
					return value == 1 ? "1 Min" : $"{value} Min";
			}
		}

		private string FormatBarsPeriodLabel(BarsPeriod bp)
		{
			if (bp == null)
				return "TF";

			if (bp.BarsPeriodType == BarsPeriodType.Minute)
			{
				if (bp.Value % 60 == 0)
				{
					int h = bp.Value / 60;
					return h == 1 ? "1 Hour" : $"{h} Hours";
				}
				return bp.Value == 1 ? "1 Min" : $"{bp.Value} Min";
			}
			if (bp.BarsPeriodType == BarsPeriodType.Day)
				return bp.Value == 1 ? "1 Day" : $"{bp.Value} Days";
			if (bp.BarsPeriodType == BarsPeriodType.Week)
				return bp.Value == 1 ? "1 Week" : $"{bp.Value} Weeks";

			return $"{bp.BarsPeriodType} {bp.Value}";
		}

		private DashStyleHelper GetDashStyle()
		{
			switch (LineStyle)
			{
				case SRLineStyleMode.Dash: return DashStyleHelper.Dash;
				case SRLineStyleMode.Dot:  return DashStyleHelper.Dot;
				default:                   return DashStyleHelper.Solid;
			}
		}

		private string MergeLabel(string existing, string add)
		{
			if (string.IsNullOrWhiteSpace(existing))
				return add ?? string.Empty;
			if (string.IsNullOrWhiteSpace(add))
				return existing;
			if (existing.Contains(add))
				return existing;
			return existing + " & " + add;
		}

		private string CreateOverlayKey(SRType type, bool ephemeral, double price)
		{
			double p = RoundToTick(price);
			return $"{type}|{(ephemeral ? 1 : 0)}|{p.ToString("0.########", CultureInfo.InvariantCulture)}";
		}

		private double RoundToTick(double price)
		{
			if (Instrument == null || Instrument.MasterInstrument == null)
				return price;
			return Instrument.MasterInstrument.RoundToTickSize(price);
		}

		private string FormatPrice(double price)
		{
			if (Instrument == null || Instrument.MasterInstrument == null)
				return price.ToString("0.########", CultureInfo.InvariantCulture);
			return Instrument.MasterInstrument.FormatPrice(price);
		}

		// Map an MTF DateTime to primary barsAgo (robust even on Renko/NinzaRenko)
		private int GetPrimaryBarsAgo(DateTime target)
		{
			int primaryLast = CurrentBars[0];
			if (primaryLast < 0)
				return -1;

			int lookback = Math.Min(primaryLast, PRIMARY_TIME_LOOKBACK);
			if (lookback <= 0)
				return 0;

			// quick bounds
			DateTime newest = Times[0][0];
			DateTime oldest = Times[0][lookback];

			if (target >= newest)
				return 0;
			if (target <= oldest)
				return lookback;

			// binary search on descending Times[0][barsAgo]
			int lo = 0;
			int hi = lookback;

			while (lo <= hi)
			{
				int mid = (lo + hi) >> 1;
				DateTime t = Times[0][mid];

				if (t == target)
					return mid;

				// t is newer than target -> need older -> move lo up
				if (t > target)
					lo = mid + 1;
				else
					hi = mid - 1;
			}

			// hi is last newer, lo is first older (or nearest)
			int idxNewer = hi;
			int idxOlder = lo;

			if (idxNewer < 0) return idxOlder;
			if (idxOlder > lookback) return idxNewer;

			double d1 = Math.Abs((Times[0][idxNewer] - target).TotalSeconds);
			double d2 = Math.Abs((Times[0][idxOlder] - target).TotalSeconds);

			return (d1 <= d2) ? idxNewer : idxOlder;
		}

		// ---------------------------
		// Properties (UI)
		// ---------------------------
		[Range(3, 50)]
		[NinjaScriptProperty]
		[Display(Name = "Pivot Length", GroupName = "General Configuration", Order = 0)]
		public int PivotLength { get; set; }

		[Range(1, 3)]
		[NinjaScriptProperty]
		[Display(Name = "Strength", GroupName = "General Configuration", Order = 1)]
		public int Strength { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Invalidation", GroupName = "General Configuration", Order = 2)]
		public SRInvalidationMode Invalidation { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Expand Lines & Zones", GroupName = "General Configuration", Order = 3)]
		public ExpandLinesMode ExpandLines { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Show Invalidated", GroupName = "General Configuration", Order = 4)]
		public bool ShowInvalidated { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "TF1 Enabled", GroupName = "Timeframes", Order = 10)]
		public bool Timeframe1Enabled { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "TF1 Unit", GroupName = "Timeframes", Order = 11)]
		public TimeframeUnit Timeframe1Unit { get; set; }

		[Range(1, 100000)]
		[NinjaScriptProperty]
		[Display(Name = "TF1 Value", GroupName = "Timeframes", Order = 12)]
		public int Timeframe1Value { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "TF2 Enabled", GroupName = "Timeframes", Order = 13)]
		public bool Timeframe2Enabled { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "TF2 Unit", GroupName = "Timeframes", Order = 14)]
		public TimeframeUnit Timeframe2Unit { get; set; }

		[Range(1, 100000)]
		[NinjaScriptProperty]
		[Display(Name = "TF2 Value", GroupName = "Timeframes", Order = 15)]
		public int Timeframe2Value { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "TF3 Enabled", GroupName = "Timeframes", Order = 16)]
		public bool Timeframe3Enabled { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "TF3 Unit", GroupName = "Timeframes", Order = 17)]
		public TimeframeUnit Timeframe3Unit { get; set; }

		[Range(1, 100000)]
		[NinjaScriptProperty]
		[Display(Name = "TF3 Value", GroupName = "Timeframes", Order = 18)]
		public int Timeframe3Value { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Show Breaks", GroupName = "Breaks & Retests", Order = 30)]
		public bool ShowBreaks { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Show Retests", GroupName = "Breaks & Retests", Order = 31)]
		public bool ShowRetests { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Avoid False Breaks", GroupName = "Breaks & Retests", Order = 32)]
		public bool AvoidFalseBreaks { get; set; }

		[Range(0.1, 2.0)]
		[NinjaScriptProperty]
		[Display(Name = "Break Volume Threshold", GroupName = "Breaks & Retests", Order = 33)]
		public double BreakVolumeThreshold { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Inverse Color After Broken", GroupName = "Breaks & Retests", Order = 34)]
		public bool InverseColorAfterBroken { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Style", GroupName = "Style", Order = 40)]
		public SRStyleMode StyleMode { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Line Style", GroupName = "Style", Order = 41)]
		public SRLineStyleMode LineStyle { get; set; }

		[Range(1, 10)]
		[NinjaScriptProperty]
		[Display(Name = "Line Width", GroupName = "Style", Order = 42)]
		public int LineWidth { get; set; }

		[Range(0.1, 10.0)]
		[NinjaScriptProperty]
		[Display(Name = "Zone Width", GroupName = "Style", Order = 43)]
		public double ZoneWidth { get; set; }

		[XmlIgnore]
		[Display(Name = "Support Color", GroupName = "Style", Order = 44)]
		public Brush SupportBrush { get; set; }

		[Browsable(false)]
		public string SupportBrushSerializable
		{
			get { return Serialize.BrushToString(SupportBrush); }
			set { SupportBrush = Serialize.StringToBrush(value); }
		}

		[XmlIgnore]
		[Display(Name = "Resistance Color", GroupName = "Style", Order = 45)]
		public Brush ResistanceBrush { get; set; }

		[Browsable(false)]
		public string ResistanceBrushSerializable
		{
			get { return Serialize.BrushToString(ResistanceBrush); }
			set { ResistanceBrush = Serialize.StringToBrush(value); }
		}

		[XmlIgnore]
		[Display(Name = "Break Color", GroupName = "Style", Order = 46)]
		public Brush BreakBrush { get; set; }

		[Browsable(false)]
		public string BreakBrushSerializable
		{
			get { return Serialize.BrushToString(BreakBrush); }
			set { BreakBrush = Serialize.StringToBrush(value); }
		}

		[XmlIgnore]
		[Display(Name = "Text Color", GroupName = "Style", Order = 47)]
		public Brush TextBrush { get; set; }

		[Browsable(false)]
		public string TextBrushSerializable
		{
			get { return Serialize.BrushToString(TextBrush); }
			set { TextBrush = Serialize.StringToBrush(value); }
		}

		[NinjaScriptProperty]
		[Display(Name = "Enable Retest Alerts", GroupName = "Alerts", Order = 60)]
		public bool EnableRetestAlerts { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Enable Break Alerts", GroupName = "Alerts", Order = 61)]
		public bool EnableBreakAlerts { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Boxes", GroupName = "Visibility", Order = 70)]
		public bool ShowBoxes { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Pane labels", GroupName = "Visibility", Order = 71)]
		public bool ShowPaneLabels { get; set; }

		[NinjaScriptProperty]
		[Display(Name = "Lines", GroupName = "Visibility", Order = 72)]
		public bool ShowLines { get; set; }
	}
}
