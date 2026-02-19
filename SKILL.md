---
name: aws-infrastructure-visualization
description: Generate FossFLOW-importable AWS Well-Architected diagram JSON that strictly passes FossFLOW zod modelSchema. Enforces required ids, icon name/url, color value, connector id, rectangle from/to, and textBoxes for labels.
license: CC-BY-4.0
compatibility: FossFLOW (stan-smith/FossFLOW) modelSchema-compatible JSON only.
metadata:
  version: "3.2"
  schema_source: "packages/fossflow-lib/src/schemas/*"
---

# FossFLOW AWS Well-Architected JSON Generator (SCHEMA-STRICT)

Return ONE raw JSON object that passes FossFLOW `modelSchema`.

## 0) Top-level required fields

- title: string (required)
- items: array (required)
- views: array (required)
- icons: array (required)
- colors: array (required)

Never output maps/objects for icons/colors; they must be arrays.

---

# 1) REQUIRED-FIELD RULE (NEVER MISSING ANY FIELD)

Every entity below MUST have `id: string`:

- icons[*].id
- colors[*].id
- items[*].id
- views[*].id
- views[*].items[*].id
- views[*].rectangles[*].id
- views[*].textBoxes[*].id
- views[*].connectors[*].id
- views[*].connectors[*].anchors[*].id
- views[*].connectors[*].labels[*].id (if labels used)

If you cannot confidently create a meaningful id, generate deterministic ids:

- icons: icon-<itemId>
- connectors: c-### and anchors c-###-a1/a2

---

# 2) Exact FossFLOW schemas (what to output)

## 2.1 icons[] (array)

Each icon MUST include:

- id: string
- name: string (REQUIRED)
- url: string (REQUIRED)
  Optional:
- collection?: string
- isIsometric?: boolean
- scale?: number (0.1..3)

## 2.2 colors[] (array)

Each color MUST include:

- id: string
- value: string (REQUIRED, max length 7; use #RRGGBB)

## 2.3 items[] (model item definitions only)

Each item MUST include:

- id: string
- name: string
  Optional:
- icon?: string (MUST match an icons[].id)

**CRITICAL RULES:**

- **NEVER include `description` field** in items[]. Keep items clean and minimal.
- items[] must NOT contain position/tile or rectangle properties.

## 2.4 views[] (layout container)

Each view MUST include:

- id: string
- name: string
- items: array of { id, tile:{x,y}, labelHeight? }

Optional:

- rectangles: array of rectangles
- connectors: array of connectors
- textBoxes: array of text boxes

**TILE SPACING RULE:**

- **Maintain minimum spacing of 2 units** between adjacent items (both x and y axes).
- Avoid placing items at consecutive coordinates (e.g., avoid x:1,y:5 and x:2,y:5; use x:1,y:5 and x:4,y:5 instead).
- Use spacing multipliers: multiply base coordinates by 1.5-2x for better visual separation.
- Example spacing: users at (0,6), next item at (4,7) or (2,8), not (1,7).

### rectangles[] (REQUIRED FIELDS: from/to)

Each rectangle MUST include:

- id: string
- from: {x:number,y:number}
- to: {x:number,y:number}
  Optional:
- color?: string (colors[].id)
- customColor?: string

DO NOT use width/height/tile/label on rectangles.

### textBoxes[] (labels)

Each textBox MUST include:

- id: string
- tile: {x:number,y:number}
- content: string
  Optional:
- fontSize?: number
- orientation?: "X"|"Y"

Use textBoxes to label VPC/subnets/AZ.

### connectors[] (REQUIRED: id + anchors)

Each connector MUST include:

- id: string (REQUIRED)
- anchors: array of anchors (REQUIRED)
  Optional:
- style?: SOLID|DOTTED|DASHED
- lineType?: SINGLE|DOUBLE|DOUBLE_WITH_CIRCLE
- showArrow?: boolean
- width?: number
- color?: string (colors[].id)
- customColor?: string
- labels?: array of {id,text,position(0..100),height?,line?,showLine?}

Anchor schema:

- id: string (REQUIRED)
- ref: object with optional fields:
  - item?: string (must reference a view item id)
  - anchor?: string
  - tile?: {x,y}

Use at least 2 anchors:

- [{ref:{item:"SRC"}},{ref:{item:"DST"}}]

---

# 3) AWS Well-Architected layout rules (VPC/AZ/Public-Private)

Default assumptions if user does not specify:

- 1 VPC, 2 AZ, public+private subnets, ALB+App+RDS

Coordinate bands:

- Edge x=1..3
- VPC x=4..18

AZ lanes:

- AZ-a base y=3
- AZ-b base y=9

Subnet zones per AZ:

- public: y=base..base+2
- private: y=base+3..base+5

Rectangles:

- VPC: from {x:4,y:2} to {x:18,y:14}
- Public AZ-a: from {x:4,y:3} to {x:18,y:5}
- Private AZ-a: from {x:4,y:6} to {x:18,y:8}
- Public AZ-b: from {x:4,y:9} to {x:18,y:11}
- Private AZ-b: from {x:4,y:12} to {x:18,y:14}

Text labels (textBoxes):

- "VPC" at (5,2)
- "Public AZ-a" at (5,3), "Private AZ-a" at (5,6), etc.

Nodes:

- route53 (1,3), cloudfront (3,3)
- alb-a (6,4), alb-b (6,10)
- app-a (11,7), app-b (11,13)
- rds-a (16,7), rds-b (16,13)

Connectors (all must have ids):

- c-001 route53 -> cloudfront
- c-002 cloudfront -> alb-a
- c-003 cloudfront -> alb-b
- c-004 alb-a -> app-a
- c-005 alb-b -> app-b
- c-006 app-a -> rds-a (dashed)
- c-007 app-b -> rds-b (dashed)

Colors must include at minimum:

- {id:"blue", value:"#0066CC"}
- {id:"orange", value:"#FF9900"}
- {id:"gray", value:"#888888"}

Icons must include name+url for every icon id referenced by items[].icon.

**AWS ICON URL RULE (MANDATORY):**

- **ALWAYS use real AWS architecture icon URLs** from the official AWS icon set.
- Base URL pattern: `https://raw.githubusercontent.com/weibeld/aws-icons-svg/main/q1-2022/Architecture-Service-Icons_01312022/`
- Icon path structure: `Arch_<Category>/64/Arch_<Service-Name>_64.svg`
- Common categories:
  - `Arch_Compute/64/` - Lambda, EC2, EC2-Image-Builder
  - `Arch_Networking-Content-Delivery/64/` - CloudFront, Elastic-Load-Balancing, Virtual-Private-Cloud
  - `Arch_App-Integration/Arch_64/` - API-Gateway, Step-Functions, Simple-Queue-Service, EventBridge
  - `Arch_Database/64/` - RDS
  - `Arch_Security-Identity-Compliance/64/` - Cognito, Secrets-Manager
  - `Arch_Management-Governance/64/` - CloudWatch, Systems-Manager
  - `Arch_End-User-Computing/64/` - WorkSpaces (for Users/Students/Admin)
- Example URLs:
  - Lambda: `Arch_Compute/64/Arch_AWS-Lambda_64.svg`
  - CloudFront: `Arch_Networking-Content-Delivery/64/Arch_Amazon-CloudFront_64.svg`
  - API Gateway: `Arch_App-Integration/Arch_64/Arch_Amazon-API-Gateway_64.svg`
- **NEVER use placeholder URLs** like `https://cdn.fossflow.io/icons/aws/...` or data URLs.
- If exact icon name is uncertain, search the weibeld/aws-icons-svg repository structure for the correct path.

Return raw JSON only.
