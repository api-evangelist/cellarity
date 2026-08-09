---
name: Read the Cellarity drug-development pipeline
description: Retrieve Cellarity's clinical and preclinical programs from the cellarity.com content API, with the caveats that keep an agent from over-reading a marketing pipeline chart as clinical data.
api: openapi/cellarity-content-openapi.yml
base_url: https://cellarity.com/wp-json/wp/v2
operations:
  - listPipelineCategories
  - listPipelineItems
  - getPipelineItem
  - listPages
  - getPage
generated: '2026-08-09'
method: generated
source: openapi/cellarity-content-openapi.yml + data-model/cellarity-data-model.yml
---

# Read the Cellarity drug-development pipeline

Cellarity's pipeline is published as the `pipeline-item` custom post type — 6 records as of
2026-08-09, backing the chart rendered at https://cellarity.com/pipeline/.

## Authentication

None. All operations here are anonymous.

## Step 1 — resolve the pipeline taxonomy

    listPipelineCategories
    GET /wp/v2/pipeline-category?per_page=100&_fields=id,slug,name,count

Observed terms: `pipeline-chart` (5), `the-details` (1), `show-popup` (1).

**Read this correctly.** `pipeline-category` encodes *presentation role* — which rows render in the
chart, which one gets the detail treatment, which triggers a popup. It is **not** a therapeutic
area, a phase, or a clinical classification. Do not map these terms onto development stage.

## Step 2 — list the programs

    listPipelineItems
    GET /wp/v2/pipeline-item?per_page=100&_fields=id,slug,link,title,content,pipeline-category

Observed programs: CLY-124 (sickle cell disease), Myelofibrosis, Exploratory Hematology, MASH,
Sickle Cell Disease, In Development.

## Step 3 — read one program

    getPipelineItem
    GET /wp/v2/pipeline-item/{id}

Read `content.rendered` — for these records the substantive detail (mechanism, stage, partner)
lives in the HTML body, not in structured fields. There are no typed fields for phase, indication
or modality on this post type. If you need those, parse the body or fall back to the newsroom
(`listNewsItems`), where clinical milestones are announced in prose.

## Step 4 — the dedicated program page

The lead asset has its own marketing page, exposed as a normal WordPress page.

    listPages
    GET /wp/v2/pages?slug=cly-124-sickle-cell&_fields=id,slug,link,title,content

    getPage
    GET /wp/v2/pages/1439

The pipeline overview page itself is `slug=pipeline` (id 321).

## Rules

- **This is a marketing surface, not a clinical register.** These six records are what Cellarity
  chose to display on a website. They are not a regulatory filing, not ClinicalTrials.gov, and not
  a complete portfolio. For trial status, go to the primary source.
- **No phase/indication/modality fields exist.** Anything you report about stage must come from
  the prose body or a dated news item — do not infer it from the taxonomy.
- **Records change silently.** `pipeline-item` is registered by Cellarity's theme, not by
  WordPress core. It can be renamed or removed by a site rebuild with no version bump, no
  deprecation notice and no changelog. Re-resolve the type from `getRouteIndex` if a collection
  starts 404ing.
- **Content fields are `{rendered: ...}` objects** containing HTML with entities. Decode before
  display.
- **Cite the `link` field**, not the API URL, when surfacing a program to a human.
- **Dates on pipeline items are CMS edit dates**, not clinical milestones. Do not present
  `date` or `modified` as a development timeline.
