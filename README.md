# A B2B Data Plane for Small Business Finance

*A fork of [Neosync](https://github.com/nucleuscloud/neosync) (YC W23, archived) re-aimed at the financial operations stack of small and mid-sized businesses and the firms, banks, and AI agents that work on top of it.*

---

## TL;DR

Small business financial data lives in eight or more disconnected systems. Every party that needs to work with it — fractional CFOs, bookkeepers, lenders, auditors, and now AI agents — either gets too much access (a privacy and compliance liability) or too little (PDFs and stale CSVs). This project repurposes Neosync's schema-aware, foreign-key-aware, deterministic-tokenization engine into a shared, agent-ready data plane that gives each party exactly the slice of an SMB's financial reality they need, with referential integrity preserved across systems.

Wedge buyer: regional outsourced accounting and fractional CFO firms. Second wedge: community banks and credit unions doing SMB lending. Long-term moat: the sanitized context layer underneath every agent that touches small-business money.

---

## The problem

A typical SMB's financial truth is scattered across QuickBooks or Xero, a primary bank (often a community bank or a neobank like Mercury or Relay), Stripe or Square, a payroll system (Gusto, Rippling, ADP Run), a bill-pay tool (Bill.com, Ramp, Brex), and a pile of spreadsheets and PDFs. The people who need to do real work against this data — the fractional CFO building a forecast, the lender underwriting a line of credit, the bookkeeper closing the month, the AI agent now trying to automate any of the above — face a binary choice: take raw access to everything (a privacy and security liability, frequently a violation of the SMB's terms with their bank or payroll provider) or work from PDFs and CSVs that are stale the moment they're generated.

This is not a hypothetical. The pain shows up in the wild constantly:

- r/Bookkeeping practitioners report sharing 1Password vaults with clients to swap login credentials, because no better workflow exists for granting scoped access to QuickBooks, Stripe, and PayPal at once.
- r/CFO and r/Accounting threads describe every new fractional engagement as a "give us access to everything" dance, with onboarding handled in Excel and Power Query because no shared tooling exists.
- r/ExperiencedDevs' canonical thread on staging environments captured the developer-side mirror of the same problem in one line: *"Everyone's data is garbage everywhere but production. This slows down development."*
- r/AI_Agents and r/LLMDevs have repeatedly hit the front page in the last sixty days with variants of *"Giving AI agents direct access to production data feels like a disaster waiting to happen,"* and r/ExperiencedFounders' *"An AI Agent Just Destroyed Our Production Data. It Confessed in Writing"* hit ninety-plus comments in a week.

The common thread is that the *connectivity* layer (Plaid, Codat, Rutter, the accounting APIs) is solved, but the layer above it — sanitized, schema-aware, permissioned, agent-ready views — is not.

## Why Neosync is the right starting point

Neosync's primitives map almost one-to-one onto what this layer needs:

- **Schema introspection** across Postgres, MySQL, MongoDB, and S3 — the foundation for understanding any SMB's data shape across systems.
- **Foreign-key-aware sync** that preserves referential integrity across tables and across databases. This is the technically hardest part of the problem, repeatedly called out in the original Neosync Show HN thread (May 2024, 246 points, 44 comments) by practitioners who had built and abandoned their own pipelines because of it.
- **Deterministic transformers** — the primitive that lets you produce stable, format-preserving tokens for the same input every time, which is what makes joins still work after sensitive fields are masked.
- **Synthetic data generation** for cases where masking is insufficient or where future-state simulation is needed.
- **Temporal-orchestrated jobs** with built-in audit trails, retries, and observability.

The codebase is archived but mature. The architectural decisions are the right ones for this problem; the work is in retargeting the connectors, transformers, and policies for SMB finance, and in rebuilding the secrets and key management story to bank-grade.

## Positioning: tokenization first, anonymization second

A common and legitimate critique of database anonymization — voiced clearly on the original Neosync Show HN — is that naive anonymization fails against re-identification: enough quasi-identifiers in aggregate (gender, geography, age band, transaction patterns) can uniquely identify a record even after names and emails are masked. This is correct, and it is doubly true in financial data, where the combination of MCC code, transaction amount, geography, counterparty pattern, and timing can uniquely fingerprint a vendor or customer.

This project does not pretend otherwise. The default posture is:

- **Deterministic, vault-backed tokenization** for any field that needs to round-trip (customer IDs, vendor IDs, account numbers, employee IDs). Real values are stored in an HSM-backed vault inside the customer's trust boundary; tokens travel through downstream systems and through model context.
- **Statistical transformations** (jitter, bucketing, k-anonymity guarantees) for fields where the value itself carries identifying signal but the distribution is what matters analytically.
- **Synthetic generation** for the cases where neither tokenization nor masking is appropriate — future-state cash flow simulation, eval data for vertical agents, demos, and onboarding sandboxes.
- **De-tokenization** happens only inside the customer's trust boundary, gated by IAM, audited per event, and never exposed to model providers.

The pitch is not "we make data anonymous." The pitch is "the model and the third party never see the real values; the real values are revealed only inside your trust boundary, only at action time, with a full audit trail."

## The product

A multi-tenant, agent-ready financial data plane. The SMB connects sources once. The platform pulls a referentially-intact copy into an isolated tenant, applies configurable transformations based on who is requesting which slice, and exposes that view to authorized parties — a fractional CFO, a lender, an auditor, a tax preparer, an AI agent, or the SMB's own copilot.

The transformations are the wedge:

- A **fractional CFO** doing cash-flow forecasting needs real amounts and real timing but not customer names or employee SSNs.
- A **lender** underwriting a line of credit needs aggregate revenue trends, deposit consistency, NSF history, and customer concentration ratios but not individual customer-level detail.
- A **bookkeeper** needs vendor names and transaction memos but not payroll detail.
- An **AI agent** building a budget needs categorized historical spend but not counterparty PII, and when it takes a real action it needs the action staged against a shadow of production before commit.

Each of these gets a different policy-driven projection of the same underlying data, with deterministic tokenization keeping joins consistent across projections.

## Why this is more than RBAC on top of Plaid

Real financial data is relational — a transaction joins to an invoice joins to a customer joins to a project joins to a payroll allocation. Strip the customer and the chain breaks; mask it naively and you can't reconcile. Neosync's FK-aware sync engine plus deterministic transformers preserve those joins across tables and across systems, so a downstream user (human or agent) can still do real work. That is the technically hard part, validated by years of practitioner pain on HN and in the dev-tooling literature, and it is exactly what Neosync's architecture solves.

The second non-obvious piece is synthetic data for *what-if* scenarios. An SMB owner asking their AI agent "what would my cash position look like if I hired two salespeople and lost my biggest customer" needs the agent to fabricate plausible future state grounded in the real historical distributions of the business. Same machinery as the anonymization layer, different output — instead of generating safe staging data, it generates safe future data, conditioned on real ledger characteristics.

## Buyer wedges

### Wedge 1: regional outsourced accounting and fractional CFO firms

The strongest wedge customer is the regional fractional CFO firm or outsourced accounting firm. These firms typically serve fifty to five hundred SMB clients, and their number-one operational pain is exactly the data-access dance described above. Reddit threads in r/Accounting and r/CFO show practitioners actively building this tooling out of duct tape today.

A deliberate note on which firms to target: the venture-backed tier of this category is unstable. Bench shut down in late 2024 and was partially absorbed by Pilot, leaving thousands of SMBs scrambling and triggering recurring r/smallbusiness and r/Bookkeeping threads about data portability and trust. Pilot itself is now the dominant survivor and represents concentration risk if a single buyer goes deep on internal tooling. The durable buyer base is the long tail of regional firms (Paro, FinOptimal, hundreds of independent and mid-sized practices) plus the emerging "AI-native accounting firm" category (Brainy and others) that absorbed Bench's refugees. Pricing is per-client-per-month, paid by the firm and passed through.

### Wedge 2: community banks and credit unions doing SMB lending

Community banks and credit unions are losing SMB lending share to fintechs (Bluevine, Enova, OnDeck, Stripe Capital, Fundbox) primarily because their underwriting takes weeks while fintechs underwrite in minutes. r/alternativelending puts the cause plainly: *"Traditional banks are built for slow, checkbox underwriting, and a lot of solid businesses get stuck waiting weeks just to hear 'no.'"* The reason fintechs are faster is direct programmatic access to borrower transaction data; the community bank still gets PDFs.

Sell the bank a *borrower data room*: the SMB authorizes a sanitized, lender-appropriate view of their financial state, refreshed continuously, with a standard set of underwriting-relevant aggregates (revenue trend, deposit volatility, customer concentration, runway, debt service coverage) computed on top. The bank gets fintech-speed data collection without ever touching raw PII; the borrower gets a faster decision and stops emailing bank statements.

A caveat: the CFPB has signaled increased scrutiny on AI in SMB lending, with explainability and fair-lending requirements coming. This product is consonant with that direction — it shortens the data-collection cycle while preserving the bank's existing decisioning model and producing a fully auditable trail of what data was used. The pitch is not "automate underwriting," it is "shorten data collection, preserve decisioning, audit everything."

## The agent context layer

This is the part that makes the play timely rather than just sensible. Every category of SMB-serving software is racing to ship agents — Intuit Assist in QuickBooks, Ramp's AI features, Brex's copilots, every vertical CFO tool's "ask anything" assistant. User reception has so far been lukewarm; r/Bookkeeping and r/QuickBooks have multiple recent high-engagement threads (*"QB AI — does anyone actually like it?"*, *"QBO AI is useless"*, *"Is QuickBooks AI just bad or am I missing something?"*) describing AI features that hallucinate categorizations, miscategorize transactions, and slow the UI down. The bottleneck is the same across all of them: the agent needs cross-system context to be useful, but giving it cross-system context means handing PII to a model provider and accepting whatever data residency and retention terms come with that.

This platform becomes the agent context layer for SMB finance. An agent — whether built by you, by the fractional CFO firm, by the bank, or by the SMB themselves — connects via MCP or a similar protocol. It gets schema-aware, permission-aware, tokenized access to the SMB's full financial state. When the agent needs to take a real action (pay a bill, move money, file a form), the action is staged against a shadow of the real data, validated, surfaced as a diff to a human or higher-trust model, and only then committed against the real systems through the appropriate connector with full audit trail.

Reddit's recent track record validates the urgency: r/AI_Agents' *"Anyone else terrified of letting agents actually do things in [production]"*, r/ExperiencedFounders' *"An AI Agent Just Destroyed Our Production Data"*, and r/technology's coverage of the Replit/Claude database deletion incident all describe the same gap this layer fills.

## Compliance and trust as a moat

Small business finance is regulated enough to be hard but not so regulated that incumbents have moats from compliance alone. GLBA applies to anyone handling consumer-ish financial data; state privacy laws (CPRA, CPA, CTDPA, and growing) apply to employee data inside payroll; SOC 2 Type 2 is table stakes; bank-grade security is required for the lending channel. Building this correctly — customer-managed encryption keys, single-tenant data isolation per SMB, deterministic tokenization with HSM-backed vaults, full audit trails on every transformation and every agent action, evidence packs for examiners — is a twelve-to-eighteen-month investment that becomes a real moat against later entrants.

Neosync's existing audit and Temporal-driven workflow scaffolding is a head start. The secrets, key management, and tenancy story has to be rebuilt to bank-grade.

## What the MVP looks like

The smallest version that is sellable:

- Connectors for QuickBooks Online, Plaid (covering most SMB banking via aggregator), Stripe, and Gusto. Connectors must be robust to Plaid's well-documented flakiness — r/QuickBooks, r/MonarchMoney, and r/OriginFinancial are full of weekly reconnect complaints, and any product layered on top of Plaid pays an engineering tax for resilience.
- A finance-tuned transformer library: vendor name canonicalization, customer tokenization that preserves concentration analysis, employee anonymization that preserves payroll aggregates, MCC-aware transaction transforms, deterministic tokenization for account and routing numbers backed by an HSM vault.
- A per-engagement workspace where a fractional CFO firm invites staff with role-based, policy-driven views.
- A query API plus an MCP server so the firm's own agents and copilots can plug in without the firm having to build the data plane themselves.
- A "share this view with my lender" flow as the second-act feature, opening the path to the bank wedge.
- Opinionated, finance-tuned defaults. The single biggest commercial signal from G2 reviews of the closest comparable product (Tonic.ai) is that setup complexity is the dominant friction. For a buyer base that is not data engineers, default to working out of the box and let advanced users customize.

With a small team forking Neosync, this is a six-to-nine-month build. From scratch it would be eighteen.

## Why now

Three shifts in the last eighteen months make this the right moment.

First, MCP and the broader agent-tool-protocol space have given a clean, standard way to expose financial data to agents without each integration being bespoke. The protocol is young enough that defining the right primitives for agent-financial-data interaction is still up for grabs.

Second, the SMB AI tooling market has exploded but user satisfaction has not. Every fractional CFO firm and every SMB-serving fintech is actively shopping for an agent data layer right now, and every incumbent's first attempt at AI features is being publicly panned on Reddit for hallucinating against fragmented context. There is no obvious winner.

Third, the regulatory environment is tightening fast enough that "the model never sees raw values" is moving from a nice-to-have to a procurement requirement, especially for anyone selling into banks. The CFPB's posture on SMB lending AI, state privacy laws on employee data, and the increasing prevalence of agent-related incidents in the press all push in the same direction.

The window to be the default data plane underneath SMB financial AI is probably eighteen to twenty-four months wide. The Neosync codebase, even archived, is a meaningful head start for whoever moves first.

## Status

This fork is a strategy and architecture document on top of an inherited codebase. Code retargeting is in progress; specific work items, roadmap, and design notes will land in /docs and in issues as they are scoped.

## Credits

Original Neosync project by Evis Drenova, Nick Zelei, and the Nucleus Cloud team (YC W23). The architectural decisions in that codebase — schema introspection, FK-aware sync, deterministic transformers, Temporal-orchestrated jobs — are the foundation this fork builds on.
