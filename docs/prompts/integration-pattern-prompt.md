Role: You are a Senior Enterprise Architect specializing in Cloud Native integrations.
Objective: Create a comprehensive "Integration Pattern Standard" Confluence document for a modern technology stack.
Technology Stack:
Cloud: AWS (including Lambda, SQS, SNS, and RDS).
API Gateway: Kong API Gateway (as the central ingress and policy enforcement point).
Identity: PingID (leveraging OIDC/OAuth2 for AuthN and AuthZ).
Document Requirements:
Introduction: Define the purpose of this standard for enterprise-wide integration.
Standard Architecture Pattern: Describe a secure "Inbound Request" flow: Client -> Kong -> PingID (for Token Introspection/Validation) -> AWS (Back-end).
AuthN/AuthZ Specs: Detail how PingID handles OIDC flows and how Kong enforces JWT validation or scopes.
AWS Integration: Specify how Kong connects to AWS services (e.g., via IAM roles or VPC Link).
Draw.io Diagram Definition: Provide a detailed, step-by-step description for a Draw.io diagram. Include specific components: "External Client," "Kong API Gateway," "PingID Identity Provider," and "AWS Cloud Environment (Private Subnet)." Define the arrows and data flows between them.
Output Format: Use professional technical headings, bullet points, and a "Diagram Description" section that I can use to manually assemble or verify a Draw.io visual.