
give me details about customer Tsehayetu

who is her account manager?

show me the account mangers using usd

what is the standard onboarding steps


find the trade name for customer email: ida@contoso.com

total sales order by ida?



please onboard: "Contoso Solutions" as as Private Company

  {
    "customerId": "CUST-1001",
    "customerName": "Contoso Solutions",
    "legalEntityType": "Private Company",
    "country": "Australia",
    "industry": "Wholesale Trade",
    "onboardingStatus": "In progress",
    "kycRiskRating": "Not assessed",
    "accountOwner": "Alex Morgan",
    "primaryContactName": "Jordan Lee",
    "primaryContactEmail": "jordan.lee@best-share-trading.example",
    "phone": "+61 2 5550 1001",
    "website": "https://best-share-trading.example",
    "annualRevenue": 4200000,
    "employeeCount": 35,
    "lastUpdatedBy": "System",
    "lastUpdatedUtc": "2024-01-10T02:00:00Z",
    "notes": "Awaiting onboarding form review."
  },


  

## Work IQ and Microsoft 365 setup status

Required:

- Temporarily activate Global Administrator for the one-time tenant setup.
- Provision the Work IQ enterprise application.
- Add delegated `WorkIQAgent.Ask` to the existing confidential web app and grant tenant-wide admin consent.
- Assign Teams Enterprise and Microsoft 365 E5 without Teams to each test user so Teams and Exchange/Outlook data are available.
- Ensure each user has Power BI/Fabric entitlement and workspace access.
- Use the native `work_iq_preview` Foundry tool with the existing project connection. Work IQ is delegated/OBO, not generic MCP or app-only authentication.

Completed and verified on 2026-08-18:

- Work IQ enterprise application provisioned.
- Existing web app has tenant-wide `WorkIQAgent.Ask` consent (`AllPrincipals`).
- Both test users have successful Teams, Exchange/Outlook, and Power BI/Fabric service plans.
- Both test users are Admins of the existing Fabric workspace.
- Existing Fabric F2 capacity resumed and verified active.
- No duplicate app registration, Foundry connection, workspace, or capacity was created.

Operator follow-up:

- Sign in to the deployed web app as each user and complete any per-user OAuth prompt.
- Run one Teams/Outlook-backed Work IQ query per user to validate delegated access end to end.
- Deactivate Global Administrator after setup.
- Pause the Fabric capacity when it is not needed to avoid ongoing cost.
- Ontology and dependent Data Agent deployment remain separate and require the applicable Fabric tenant preview settings.