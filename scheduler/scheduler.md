### There are two supported way to configure the filtering and scoring behavior of the scheduler

1. **Scheduling Policies** allow you to configure Predicates for filtering ans Priorities for scoring.
2. **Scheduling Profiles** allow you to configure Plugins that implement different scheduling stages, including: QueueSort, Filter, Score, Bind, Reserve, Permit, and others. You can configure the kube-scheduler to run different profiles.

