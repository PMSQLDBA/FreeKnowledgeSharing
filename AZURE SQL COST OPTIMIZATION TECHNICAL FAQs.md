AZURE SQL COST OPTIMIZATION TECHNICAL FAQs

BEGINNER LEVEL

Q1: What are the two main purchasing models for Azure SQL Database?
- DTU-based model
  * Bundles CPU, memory, and I/O into single blended unit
  * Fixed price per tier
- vCore-based model
  * Choose CPU cores and memory independently
  * Unlocks Azure Hybrid Benefit for reusing existing SQL Server licenses
  * More cost-flexible for production workloads
  * Allows scaling each dimension separately
  * Avoids paying for bundle portions you don't need

Q2: How does Azure Hybrid Benefit reduce Azure SQL costs?
- Applies existing on-premises SQL Server licenses toward Azure SQL compute cost
- Requires active Software Assurance on licenses
- Provides up to 40% savings on compute costs
- Exact savings depends on specific licensing and tier
- Available only on vCore model, not DTU
- Applies licensing you already own and likely already paying for

Q3: What is a reserved capacity purchase and why does it save money?
- Commit to specific vCore amount for 1-year or 3-year term
- Pay upfront or monthly
- Get meaningfully lower rate than pay-as-you-go pricing
- Trade-off: lose flexibility due to capacity commitment
- Suitable for steady-state, predictable production workloads
- Not ideal for spiky or short-lived workloads

Q4: How does the serverless compute tier work and when does it save money?
- Auto-scales compute based on actual load
- Can auto-pause during genuine inactivity
- Bills per-second for what's actually used
- Not fixed reserved amount billing
- Saves money for unpredictable, intermittent usage
- Best for dev/test databases
- Good for new low-traffic applications
- Avoids paying idle capacity costs

Q5: Why might serverless NOT save money for steady, always-on production workloads?
- Consistent, predictable load means per-second billing becomes expensive
- Often more expensive than reserved or standard provisioned tier
- Cost advantage comes from variability and idle time
- Steady-state workloads lack these advantages
- Not suitable for continuous, stable usage patterns

Q6: What is an elastic pool and how does it help control costs for many databases?
- Shared resource pool for multiple databases
- Databases draw from pool instead of individual allocations
- Cost-effective for databases with unpredictable, non-overlapping usage spikes
- Pay for pool's shared peak capacity
- More economical than sum of individual database peaks
- Works best when usage patterns don't overlap

Q7: What's the risk of leaving a database provisioned at a higher service tier than it actually needs?
- Paying for compute and capability workload doesn't use
- Very common cost drain
- Easy to overlook
- Databases often sized generously "to be safe" during setup
- Never revisited after initial deployment
- Real usage patterns often much lower than provisioned capacity

Q8: What is Azure Advisor and how does it help with cost optimization?
- Includes cost recommendations category
- Flags databases with sustained low utilization
- Suggests downsizing opportunities
- Identifies workloads benefiting from reserved capacity
- Based on consistent historical usage analysis
- Surfaces savings opportunities otherwise missed

Q9: Does Azure SQL Database storage cost scale linearly with actual data or configured maximum size?
- Storage cost generally based on actual data consumed
- Not based on maximum size limit configured
- Important caveat: some bundled-storage scenarios differ
- Certain service tiers include storage bundles
- Behavior varies between pure consumption billing
- Review specific tier's billing model

Q10: What is backup storage cost and how can it add up unexpectedly?
- Backup storage beyond included amount billed separately
- Geo-redundant storage (GRS) roughly doubles backup footprint
- Locally redundant storage (LRS) is cheaper alternative
- GRS maintains full copy in another region
- Cost factor easy to overlook
- Not part of headline compute pricing

Q11: How does Long-Term Retention affect cost?
- LTR backups billed separately from standard retention
- Billing based on amount retained and duration
- Not included in standard short-term backup retention
- Aggressive, unreviewed multi-year policies become costly
- Often applied broadly without genuine compliance need
- Can become meaningful, overlooked cost line over time

Q12: What's a simple first step for reviewing Azure SQL costs across a subscription?
- Use Azure Cost Management plus Billing
- Get breakdown of spend by resource
- Quickly identify which databases drive largest cost portion
- Natural starting point before individual optimizations
- Enables proper prioritization of optimization efforts

Q13: Can you scale an Azure SQL Database down during off-peak hours to save money?
- Technically possible via scripting or automation
- Use PowerShell, Azure Automation, or Logic Apps
- Scale down during predictable low-traffic windows
- Scale back up before peak periods
- Legitimate cost-saving pattern for predictable cycles
- Best for daily or weekly usage patterns
- Adds operational complexity
- Not worth it without clear, reliable low-usage window

Q14: What's the cost difference between General Purpose and Business Critical tiers?
- Business Critical costs meaningfully more
- Includes additional continuously running replicas
- Fully synchronized replicas with local storage
- Faster HA failover capability
- General Purpose cheaper due to simpler HA model
- General Purpose uses compute-storage-reattachment HA
- Business Critical maintains live redundant copies

Q15: Is it cheaper to run many small, separate Azure SQL databases or consolidate them into an elastic pool?
- Depends entirely on usage patterns
- Unpredictable, non-overlapping peaks: pooling cheaper
- Sharing capacity beats individual worst-case provisioning
- Consistently near-idle or near-peak at same time: smaller savings
- Savings can even reverse with similar patterns
- Requires analysis of actual workload patterns

Q16: What's the cost implication of choosing Hyperscale for a database that doesn't actually need very large storage?
- Hyperscale priced competitively for offerings
- Real value for large or fast-growing databases
- Storage architecture benefit for scale
- Fast backup/restore matters for large databases
- Small, simple database may not get proportional value
- Compare against simpler tier for non-scaling needs

Q17: What's a simple habit that helps prevent cost creep over time?
- Periodically review actual resource utilization
- Compare against provisioned capacity
- Do this not just at setup, but recurring basis
- Workloads change and sizing assumptions age
- Database correctly sized year ago may no longer be correct
- Drift can occur in either direction

INTERMEDIATE LEVEL

Q18: How would you quantify whether a specific database is a good candidate for downsizing?
- Pull sustained CPU/DTU utilization trends
- Use Azure Monitor metrics or sys.dm_db_resource_stats
- Look for weeks of data, not just days
- Capture real variability including periodic peaks
- Database averaging 20% but spiking to 90% for critical job: different candidate than never exceeding 30%
- Consider peak utilization, not just average
- Ensure capture includes all usage patterns

Q19: What's the difference in cost and benefit between reserved capacity and Azure Hybrid Benefit?
- Address different cost levers
- Generally can combine both
- Reserved capacity: discounts compute rate with term commitment
- Azure Hybrid Benefit: reduces licensing-inclusive rate with existing licenses
- Using both together: deepest combined discount
- Best for steady-state, license-eligible production workloads
- Not mutually exclusive savings mechanisms

Q20: How would you decide between 1-year and 3-year reserved capacity terms?
- Consider confidence in workload stability
- 3-year term offers deeper discount
- 3-year locks in assumption about future needs
- Riskier for growing or changing applications
- Better for mature, genuinely stable workloads
- 1-year terms for uncertain future sizing
- Account for real uncertainty in planning horizon

Q21: What's the cost risk of over-provisioning an elastic pool's total eDTU or vCore capacity?
- Pay for pool's total provisioned capacity regardless of usage
- Generously-sized pools "to be safe" become wasteful
- Same waste as over-provisioned single database
- Spread across multiple databases can hide waste
- Waste less obvious when hidden across many databases
- Base sizing on actual aggregate peak usage data

Q22: How would you use Query Performance Insight or Query Store data in a cost optimization review?
- Resource-consuming query drives compute capacity needs
- Inefficient query requires larger database sizing
- Fixing query enables potential downsize
- Downsize may not be safe with unoptimized query running
- Look for obvious tuning opportunities in top queries
- Query optimization is legitimate cost lever
- Sometimes enables tier reduction

Q23: What's the cost implication of enabling verbose diagnostic logging categories across a large fleet?
- Log Analytics ingestion billed per GB
- Chatty categories on busy databases accumulate
- Costs add up meaningfully across large fleet
- Be deliberate about categories needed at full fidelity
- Production can run full categories; non-prod can reduce
- Consider Basic Logs tier for high-volume categories
- Don't default every database to full Analytics logging

Q24: How would you evaluate whether serverless is actually cheaper than provisioned compute for a specific workload?
- Model both options against historical usage pattern
- Compare serverless per-second billing rate
- Include real hours of active use in calculation
- Factor in any auto-pause savings
- Compare directly against provisioned tier's flat rate
- Use Azure pricing calculator with real data
- Don't rely solely on general guidance

Q25: What's the relationship between right-sizing and read scale-out from a cost perspective?
- Peak consumption driven partly by reporting/analytical queries
- Route reporting traffic to read-only secondary
- Reduces primary's peak load
- May justify smaller primary tier
- Read scale-out changes what primary needs sizing for
- Genuine cost lever, not just performance benefit
- Included at no extra cost on certain tiers

Q26: How would you approach cost optimization for backup storage without compromising recovery requirements?
- Separate genuine compliance-driven LTR needs
- Identify default settings nobody has revisited
- Right-size LTR retention granularity
- Match to actual requirements, not most generous option
- Evaluate GRS necessity for each database
- Some databases can reasonably use LRS instead
- Document deliberate trade-offs against DR requirements

Q27: What's the cost consideration around database consolidation?
- Combined workload doesn't need sum of separate provisions
- Can reduce total cost
- Trade-off: added complexity
- Shared blast radius if one workload has problem
- Potential noisy-neighbor effects within same database
- Weigh savings against introduced complexity
- Not a free cost reduction

Q28: How would you handle cost optimization for non-production environments?
- Usually easiest, lowest-risk place for savings
- Use smaller service tiers for dev/test load
- Apply serverless with auto-pause for idle environments
- Use shorter backup retention where compliance allows
- No need to match production retention policies
- Greatest room for aggressive optimization

Q29: What's a cost-related reason to periodically review Automatic Tuning's applied actions?
- Automatically created indexes may not deliver proportional benefit
- Index adds write overhead and storage cost
- Return may not justify overhead
- Automatic Tuning designed to self-correct performance issues
- Cumulative index footprint builds over time
- Worth reviewing from cost and efficiency perspective

Q30: How would you decide whether a specific database genuinely needs Business Critical tier?
- Tie decision explicitly to actual RTO requirement
- Consider real read scale-out need
- Not based on general "this feels important" instinct
- General Purpose HA model often sufficient
- Database may not need free read-only replica
- Business Critical's extra cost may not align with actual business need

Q31: What's the cost impact of choosing geo-redundant backup storage across an entire fleet by default?
- GRS roughly doubles backup storage cost vs LRS
- Applied uniformly across fleet becomes expensive
- Lower-criticality databases may not need cross-region DR
- Paying doubled cost for protection not genuinely required
- Typical blanket-default cost creep
- Worth reviewing and tiering by actual criticality

Q32: How would you approach cost optimization when a team resists downsizing due to past performance incidents?
- Validate recommendation with actual data
- Show real sustained utilization trends
- Propose conservative, monitored downsize
- Offer easy rollback path
- Scaling back up takes minutes, not major project
- Make decision evidence-based and reversible
- Easier sell than all-or-nothing framing

Q33: What's the cost consideration around monitoring and diagnostic data itself?
- Log Analytics workspace can cost more than databases monitored
- Especially when retention and verbose categories accumulate
- Genuinely common blind spot
- Can happen if left unreviewed
- Compare monitoring spend against database spend
- Treat disproportionate monitoring cost as optimization target
- Not just assumed cost of doing business

Q34: How do you approach the trade-off between cost savings and operational risk?
- Weigh actual savings against introduced complexity
- Consider failure mode risks
- Automated scale-down/scale-up adds new failure point
- Scale-up not happening in time before peak traffic: real risk
- Savings must justify added operational risk
- Must justify maintenance burden
- Not pursued just because technically available

Q35: What's a practical way to build ongoing cost awareness into a DBA team's workflow?
- Fold lightweight utilization and cost review into regular cadence
- Include in monthly ops review instead of separate initiative
- Review Advisor's cost recommendations
- Check for configuration drift from original sizing
- Revisit reserved capacity coverage as workloads change
- Treat as routine maintenance
- Not a special project for occasional attention

ADVANCED LEVEL

Q36: Design a comprehensive, ongoing cost optimization program for a large Azure SQL fleet (500+ databases) across multiple business units.
Strategy:
- Establish visibility through tagging
  * Tag by business unit
  * Tag by environment
  * Tag by criticality
  * Enables cost attribution and prioritization
- Implement tiered review cadence
  * Automated weekly reports flagging outliers
  * Quick action items from weekly analysis
  * Quarterly deeper review covering:
    . Reserved capacity coverage against actual usage
    . Azure Hybrid Benefit eligibility gaps
    . Architecture-level reconsideration
- Assign clear ownership
  * Assign accountability for acting on findings
  * Reports without action ownership fail
  * Someone must own implementation

Q37: How would you build a data-driven business case for migrating databases to 3-year reserved capacity?
- Pull utilization history
  * Minimum 6-12 months per database
  * Validates steady-state, not assumptions
- Calculate actual discount
  * Real committed vCore amount
  * Current pay-as-you-go spend comparison
  * Specific numbers for legitimacy
- Model downside scenario
  * What if meaningful subset needs to scale down
  * What if databases get decommissioned mid-term
  * ROI impact of adverse events
  * Honest representation of commitment risk
  * Earns trust for multi-year recommendation

Q38: Explain the cost and architecture trade-off between Hyperscale and General Purpose for a rapidly growing database.
Analysis framework:
- Evaluate Hyperscale against growth trajectory
  * Not just current size
  * Growth pattern matters significantly
- General Purpose storage cost scaling
  * Becomes less favorable as database grows
  * Backup/restore time increasingly costly
  * Operational risk grows with size
- Hyperscale benefits
  * Architecture keeps operations fast regardless of size
  * Backup/restore remains manageable
- Compare over multi-year horizon
  * Model total cost of ownership
  * Realistic growth projections
  * Don't rely on point-in-time comparison

Q39: How would you approach cost optimization for a multi-tenant SaaS application on Azure SQL?
Key challenge:
- Tenant-level cost attribution often blocked by architecture
- Cannot optimize what you cannot attribute
Architectural evaluation:
- Single large database: hard tenant attribution
- Elastic pool: pooling efficiency vs attribution challenge
- Per-tenant databases: easy attribution, lose pooling efficiency
Recommended hybrid approach:
- Pool small, low-usage tenants together
  * Gain pooling efficiency
  * Minimal attribution need for small tenants
- Isolate largest tenants in own databases
  * Clear cost attribution per tenant
  * Prevent large tenant from driving pool sizing
  * Accurate cost-per-tenant billing if needed

Q40: What's your approach to evaluating whether Azure Hybrid Benefit is being fully utilized across a large fleet?
Process:
- Audit actual license entitlement
  * SQL Server licenses with active Software Assurance
  * Organization actually owns and pays for
- Compare against applied licenses
  * Current application across vCore fleet
  * Common drift points:
    . New databases provisioned without benefit
    . Licenses covering more vCores than applied
    . Licenses applied to wrong tiers
- Characteristics of this review
  * Highest-value cost opportunity
  * Lowest-risk cost opportunity
  * No capacity reduction required
  * No performance impact
  * Just applying already-owned entitlements
  * Organization likely already paying for licenses separately

Q41: How would you design cost-aware autoscaling for a workload with a predictable daily usage pattern?
Foundation:
- Base schedule on real captured historical data
  * Not assumptions
  * Actual usage patterns
- Build in safety margin
  * Scale-up happens before peak load begins
  * Comfortable margin, not at boundary
  * Avoid discovering inadequate sizing when already under load
- Implement comprehensive monitoring
  * Monitoring specifically on automation itself
  * Confirm scale operations complete successfully
  * Catch failed scale-up before it causes impact
- Include manual overrides
  * Easy override for unexpected off-schedule demand
  * Rigid schedule cannot accommodate genuine unplanned peak
  * Flexibility for exceptions

Q42: Explain how to quantify the hidden cost of technical debt in Azure SQL sizing decisions.
Process:
- Run fleet-wide sustained-utilization analysis
  * Flag databases over-provisioned vs actual usage
  * Current provisioned tier vs sustained peak usage
- Calculate delta
  * Gap between current spend and right-sized spend
  * Aggregate across entire fleet
  * Put real number on "never revisited" cost
- Frame for organizational impact
  * Present as ongoing, compounding monthly cost
  * Not one-time number
  * Much more persuasive for action
  * Shows true hidden cost

Q43: How would you approach the cost/performance trade-off when a proposed optimization carries genuine risk?
Methodology:
- Make trade-off explicit
  * Not framed as purely technical decision
  * Not hidden inside "just save money" messaging
- Quantify clearly
  * Here's the monthly savings
  * Here's the specific, concrete risk accepted
  * Example: slower failover time, narrower performance headroom margin
- Get conscious approval
  * From whoever owns business risk
  * Not DBA-unilateral decision
  * Document the approval
- Risk management principle
  * Cost optimization trading away resilience silently causes expensive incidents
  * Informed decision prevents hidden risk
  * Goal is awareness, not just smaller bill

Q44: What's your approach to evaluating cost optimization opportunities that require code or application-level changes?
Prioritization framework:
- Weigh engineering effort
  * Deployment risk associated with code changes
  * Cost payoff of optimization
  * Any performance benefit gained
- Prioritize configuration-level optimizations first
  * Lower risk
  * Faster to realize
  * No code deployment needed
- Track code-level opportunities as backlog
  * Longer-term implementation
  * Coordinate with application team
  * Not expecting DBA to unilaterally drive
  * Example: genuinely wasteful query enabling downsize
- Realistic expectations
  * Code changes require application team buy-in
  * DBA cost optimization cannot drive application changes alone
  * Requires cooperation and timing alignment

Q45: How would you design cost governance guardrails to prevent cost creep at the point of provisioning?
Policy implementation:
- Enforce tagging requirements
  * Every new database attributable from day one
  * Makes cost tracking possible from start
  * Prevents blind spots later
- Restrict service tier provisioning
  * Require approval for tiers above defined threshold
  * Prevents unnecessary high-tier defaults
  * Catches over-provisioning before it starts
- Standardize diagnostic settings
  * Retention policies follow approved defaults
  * Not configured ad hoc per database
  * Prevents inconsistent, excessive logging
- Shift discipline earlier
  * Prevents over-provisioning at creation time
  * Cheaper than correcting months later
  * Less disruptive than changing existing databases
  * Workload dependencies haven't grown around oversize baseline

Q46: Explain the cost consideration of Automatic Tuning's CREATE INDEX option at fleet scale.
Cost accumulation pattern:
- Individual index impact
  * Marginal storage cost
  * Marginal write overhead
- Fleet-scale accumulation
  * Hundreds of databases
  * Each accumulating tuning-driven indexes
  * Months or years without holistic review
  * Marginal costs compound into significant sum
- Monitoring and review
  * Include index count trends in reviews
  * Track total index storage footprint
  * Part of periodic fleet-wide cost review
  * Otherwise slow, distributed cost creep remains invisible
  * No single database's bill makes it obvious

Q47: How would you handle cost optimization recommendations that conflict with compliance or security requirements?
Distinction principle:
- Genuinely mandated costs: non-negotiable
  * Cannot reduce these directly
  * Compliance requirements non-discretionary
- Default or conventional costs: discretionary
  * Legitimate optimization targets
  * May not have real mandate behind them
Approach for mandated costs:
- Don't quietly reduce them
- Ensure mechanism is itself efficient
  * Right retention granularity
  * Right storage tier for archival data
  * Right redundancy level for actual requirement
- Optimize how requirement is delivered
  * Not reduce the requirement itself
  * Different from trying to avoid requirement

Q48: What's your approach when a business unit resists any change to their database configuration?
Strategy:
- Start with low-risk pilot
  * Less business-critical database in their environment
  * Reversible change
  * Demonstrates safety in practice
- Build trust through demonstrated success
  * Real, positive track record
  * Confidence in your judgment
  * Before proposing changes to sensitive systems
- Apply pilot results
  * Reference successful pilot
  * Show lack of issues
  * Easier sell to critical systems
- Recognize psychology
  * Trust rebuilt through demonstrated safe track record most effective
  * Better than purely analytical case
  * Especially after past incident
  * Experience beats arguments

Q49: How would you measure and report the ROI of a cost optimization program over time?
Methodology:
- Track actual realized savings
  * Against documented baseline
  * What fleet would have cost without optimization actions
  * Not just theoretical savings from time of recommendation
- Account for implementation effects
  * Savings estimated at recommendation time may diverge from actual
  * Downsize later needs to scale back up for legitimate reasons
  * Don't count as permanent saving in ongoing report
- Report reversals honestly
  * Any optimization actions walked back
  * Reasons they were reversed
  * Impact on overall ROI
- Leadership accuracy
  * Program only reporting successes not giving accurate picture
  * Financial credibility depends on honest reporting
  * Trade-offs must be visible to leadership

Q50: Walk through a genuinely difficult cost optimization challenge end to end.
Universal strong structure regardless of specific scenario:
1. Establish real data first
   * Actual utilization metrics
   * Actual usage patterns
   * Not assumption or quick portal glance
   * Multi-week trending, not single data points
2. Identify genuine trade-off
   * Every saving costs something somewhere
   * Make it explicit, not hidden
   * Performance headroom trade-off
   * Resilience trade-off
   * Flexibility trade-off
3. Prioritize low-risk, reversible changes
   * Especially when building process trust
   * High-risk changes come later
   * Demonstrate competence first
4. Get conscious risk approval
   * Whoever owns associated risk
   * Not DBA unilateral decision
   * Document the approval
   * Formal risk acceptance
5. Measure and report actual results
   * Include reversals and walked-back changes
   * Real realized savings vs estimated
   * Don't declare victory at implementation
   * Follow-up measurement proves value

ADDITIONAL CONSIDERATIONS

Core principle:
- Cost optimization not about finding one clever trick
- Disciplined, ongoing, evidence-based review
- Honest about what every saving costs elsewhere
- Confident enough to say plainly when opportunity not worth its cost

Key principles for cost optimization:
- Data-driven decisions using actual utilization trends, not assumptions
- Evidence-based trade-off analysis between cost, performance, and resilience
- Recurring, routine reviews rather than one-time optimizations
- Clear accountability and ownership for implementing recommendations
- Reversible changes prioritized over irreversible ones
- Honest reporting of both successes and reversals
- Visibility and attribution through proper tagging and monitoring
- Right-sizing as the foundation for all other optimizations
- Coordination with application teams for code-level improvements
- Distinction between mandatory and discretionary costs

Cost factors often overlooked:
- Backup storage costs with geo-redundancy
- Diagnostic logging ingestion at fleet scale
- Automatic tuning index accumulation
- Reserved capacity flexibility trade-offs
- Monitoring costs disproportionate to database costs
- Long-term retention policies exceeding genuine requirements
- Database configuration drift over time
- Unused elastic pool capacity
- Oversized databases sized for safety rather than necessity
