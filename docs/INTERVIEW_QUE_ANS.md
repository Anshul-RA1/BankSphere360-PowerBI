## Day 1 — Requirements & Lifecycle

### Q: Walk me through your BI development lifecycle.

**Chain:** requirements → source analysis → data model → ETL → measures → visuals → security → deploy → support

**Spoken answer:**
"I follow a consistent lifecycle. First I gather requirements — who the
users are and what decisions they need to make. Then source analysis:
what systems hold the data, the grain, and the quality. Next I design the
data model — usually a star schema — before touching visuals, because a
clean model makes everything downstream easy. Then ETL in Power Query to
cleanse and shape, followed by the DAX measures for the KPIs. Only then do
I build visuals. After that comes security — Row-Level Security — then
deployment with scheduled refresh, and finally ongoing production support.
On BankSphere 360, for example, I froze eight KPIs with stakeholders before
I built anything, which kept the whole build on track."

**Why this order matters (if pushed):** skipping the model step and jumping
to visuals is the classic mistake — you end up patching logic across 40
visuals instead of fixing it once in the model.

---

### Q: How do you gather requirements?

**Keywords:** personas · KPI definitions signed off in writing · mock-up first · avoid scope creep

**Spoken answer:**
"I start with stakeholder workshops to identify personas — on BankSphere 360
that was five: leadership, branch heads, risk, relationship managers, and
compliance. For each persona I capture the decisions they need to make, then
translate those into specific KPIs with agreed definitions. The key step is
getting those KPI definitions signed off in writing — for instance, exactly
how NPA% is calculated — so there's no dispute later. I then build a quick
mock-up to confirm the layout before development. Anything new that comes up
after sign-off becomes a logged change request, which is how I avoid scope
creep."

**Senior signal:** the written KPI sign-off is the detail that separates a
6-year professional from a junior — it prevents the "three teams, three
numbers" problem.
