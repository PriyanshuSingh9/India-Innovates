# Intelligence Products

> Covers: F31 Strategic Brief Generator

---

## F31 — Strategic Brief Generator

### What Is Being Built

An automated system that generates comprehensive intelligence briefs every 6 hours by synthesizing recent events, BII scores, anomaly alerts, and convergence signals into a structured, LLM-generated document. The brief provides Indian strategic personnel with a "morning report" covering global developments through India's strategic lens.

### Why It Is Needed

Writing intelligence briefs manually takes hours. Analysts read hundreds of reports, cross-reference sources, and synthesize findings into a coherent document. BIE automates this by:
- Aggregating the last 6 hours of events across all theaters
- Weighting by BII scores and India-relevance
- Detecting patterns (convergence, anomalies)
- Using Claude API to synthesize a readable briefing document

This transforms BIE from a dashboard into an **intelligence production system**.

### What Use It Provides

- **Auto-generated every 6 hours** — always-current intelligence summary
- **Global scope with India impact** — covers all theaters, explicitly analyzes India implications
- **Structured format** — consistent sections for easy scanning
- **Event citations** — every claim links back to source events
- **Posture assessment** — current threat posture with trend analysis
- **Watch items** — upcoming events and developing situations to monitor
- **Exportable** — download as PDF or Markdown

### How It Is Built

1. **Brief sections**:
   ```python
   BRIEF_TEMPLATE = {
       "sections": [
           {
               "title": "Executive Summary",
               "prompt": "Provide a 3-paragraph executive summary of the most significant global developments in the past 6 hours, focusing on their implications for Indian national security and strategic interests."
           },
           {
               "title": "India Neighborhood Watch",
               "prompt": "Analyze developments in India's immediate neighborhood (Pakistan, China border, Sri Lanka, Nepal, Bangladesh, Myanmar). Highlight any changes in military posture, diplomatic signals, or security incidents."
           },
           {
               "title": "Indo-Pacific Theater",
               "prompt": "Cover key developments in the Indo-Pacific: South China Sea, Taiwan Strait, East China Sea, Korean Peninsula, ASEAN dynamics. Assess implications for India's Act East Policy and QUAD."
           },
           {
               "title": "Middle East & Energy Security",
               "prompt": "Analyze events in the Middle East affecting India: Persian Gulf stability, Red Sea shipping, energy supply disruptions, diaspora security, India-Gulf state relations."
           },
           {
               "title": "Global Strategic Developments",
               "prompt": "Cover significant developments in Africa (BRI, security), Central Asia (SCO, Afghanistan), Europe (NATO, Russia), and Americas affecting India's interests."
           },
           {
               "title": "Threat Assessment",
               "prompt": "Provide a structured threat assessment showing BII scores for key regions, active anomalies, and convergence alerts. Rate overall strategic posture."
           },
           {
               "title": "Watch Items",
               "prompt": "List 3-5 developing situations that require monitoring over the next 24-48 hours. Include upcoming events (summits, exercises, elections) that may impact India."
           }
       ]
   }
   ```

2. **Data gathering** — before calling Claude:
   ```python
   class BriefGenerator:
       async def generate(self):
           # Gather context
           events = await self.get_events(hours=6)
           bii_scores = await self.get_bii_all_regions()
           anomalies = await self.get_active_anomalies()
           convergence = await self.get_active_convergence()
           top_entities = await self.get_trending_entities(hours=6)

           # Build brief
           sections = []
           for section in BRIEF_TEMPLATE["sections"]:
               context = self._build_section_context(
                   section, events, bii_scores, anomalies, convergence, top_entities
               )
               content = await self.claude.generate(
                   system="You are a senior intelligence analyst advising Indian strategic leadership. "
                          "Write concise, actionable intelligence briefs. Cite event IDs where possible.",
                   prompt=f"{section['prompt']}\n\nContext:\n{context}"
               )
               sections.append({"title": section["title"], "content": content})

           return {
               "id": str(uuid.uuid4()),
               "generated_at": datetime.utcnow().isoformat(),
               "period_start": (datetime.utcnow() - timedelta(hours=6)).isoformat(),
               "period_end": datetime.utcnow().isoformat(),
               "sections": sections,
               "instability_scores": bii_scores,
               "overall_posture": self._calculate_posture(bii_scores),
           }
   ```

3. **Posture calculation** from brief data:
   ```python
   def _calculate_posture(self, bii_scores):
       max_score = max(s["score"] for s in bii_scores.values())
       if max_score > 0.8:
           return "CRITICAL"
       elif max_score > 0.6:
           return "HIGH"
       elif max_score > 0.4:
           return "ELEVATED"
       return "NORMAL"
   ```

4. **Scheduling**: runs every 6 hours via APScheduler. Latest brief cached in Redis (`bie:cache:brief:latest`, 6h TTL). Also triggerable manually via `POST /api/brief/generate`.

5. **Cost optimization**: each brief makes ~7 Claude API calls (one per section). Claude responses are cached aggressively. Total cost per brief: ~$0.50–$1.00. Daily cost: ~$2–$4.

---

← Back to [prd.md](./prd.md)
