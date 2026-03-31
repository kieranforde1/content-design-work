# DIST-017 | Email Signature Generator Copy Modal

| Slack Channel | [\#dist-017-esg-copy-modal](https://hubspot.slack.com/archives/C04A8DS8SCQ) |
| :---- | :---- |
| **DRI(s)** | [Adam Stevenson](mailto:astevenson@hubspot.com), [Conor Heffernan](mailto:cheffernan@hubspot.com) |
| **Status** | Complete / Successful |
| **KPIs** | **North Star:** Signups completed **Primary:** Signups initiated **Guardrail:** Leads generated |
| **Readout Deck** | Link [here](https://docs.google.com/presentation/d/1OnePyn_zZYx0LNi3QQnoLUVgUaXxRnvTzoNycNo5X44/edit#slide=id.g11da112551b_0_303) |
| **Results** |  |

# Problem

HubSpot’s [Email Signature Generator](https://www.hubspot.com/email-signature-generator) (ESG) is currently our [most visited microapp](https://analytics.amplitude.com/hubspot/chart/lng8bn3), regularly receiving 4 times as many pageviews as our next most popular app. Yet, as is the case for almost all of HubSpot’s existing microapps, ESG drives a relatively [insignificant number of monthly signups](https://analytics.amplitude.com/hubspot/chart/6wil95u) to the HubSpot platform. We believe this is because signup motions are not prominently featured after a user has completed the creation of their email signature.

# Objective

Our primary objective is to **increase the number of users starting and completing signup** in the Email Signature Generator app from a new modal that appears after a user copies their new email signature.

# Hypothesis

We believe that there are users who would be interested in signing up for HubSpot after completing the creation of their email signature, but there are insufficient CTAs to prompt users to sign up after completing the email signature flow.

We believe this because:

* Only [\~450 signups are initiated within 1 day](https://analytics.amplitude.com/hubspot/chart/03bfwkr) out of \~16k weekly visits to the app (2.6%).  
* All signup CTAs appear **below the fold** on the most common display viewport sizes.

![][image1]

## Typical viewport cutoff

## **Prediction**

**(Primary)** We believe that we can **increase signups initiated by 1.4pp to 4.0%** from Email Signature Generator by adding a new modal that appears after a user copies their new email signature that prompts users to sign up for HubSpot. We predict this prompt will lead to **\~250** more signups initiated **per week** (from 450 to 700).

**(North Star)** We believe signups completed will increase with the increased number of signups initiated, but we do not believe that the conversion from Signup Initiated → Signup Completed will be affected by this experiment.

**(Guardrail)** We believe that [lead generation](https://looker.hubspotcentral.net/looks/68001?toggle=dat,fil&qid=orsLi45vAeJf6EalOW3eij) from ESG will not decrease if we add a CTA to the header.

# Experiment Design

[Figma Mockups](https://www.figma.com/file/lwdsXoLSbWrjXwtJTzoZLM/Email-Signature-Generator-Copy-Modal?node-id=9%3A34&t=GkODdUaIBcazXBNA-3)

| Sample Criteria | Users who reach the final page of Email Signature and click either Copy signature or Copy signature source code |
| :---- | ----- |
| **Control Group** | (Existing UX) When a user clicks one of the **Copy** options, a **Copied\!** floating alert appears in the top right and the email signature content is copied to the user’s device clipboard **Exposure:** 33.3…% ![][image2] Existing copy UX with **Copied\!** alert |
| **Variant Group A** | When a user clicks one of the **Copy** options, a confirmation modal appears, containing: Positive confirmation that the email signature generator was successfully copied A prompt to sign up for HubSpot and a CTA that links to the signup flow in a new tab **Exposure:** 33.3…% *![][image3]* Modal with signup CTA that appears when clicking one of the “Copy” buttons |
| **Variant Group B** | When a user clicks one of the **Copy** options, a confirmation modal appears, containing: Positive confirmation that the email signature generator was successfully copied A prompt to sign up for HubSpot and a CTA opens an embedded version of the signup flow inside the modal **Exposure:** 33.3…% *(Mockups WIP)* |
| **Experiment Runtime** | 2 weeks |

# Metrics

***All metrics assume the experiment will run for 14 days.***

## **Signups Initiated from ESG final page (Primary)**

[Data Source](https://analytics.amplitude.com/hubspot/chart/6wjvm4u) (Amplitude) | [Significance calculation](https://tools.hubteam.com/significance/calculator?confidenceThreshold=0.05&vC-0=48&vE-0=2977&vC-1=119&vE-1=2977&vC-2=119&vE-2=2977&vN-0=Control&vN-1=Variation+A&vN-2=Variation+B)

| Metric | Control | Variants A & B (Predicted) |
| :---- | :---- | :---- |
| ESG (download page) Pageviews | 2,977 (33.3…% exposure) | 2,977 (33.3…% exposure) |
| Signups Initiated | 48 | 119 (×2\) |
| **Signups Initiated CVR** | **1.6%** | **4.0%** |

## **Signups Initiated from ESG first page (Secondary)**

[Data Source](https://analytics.amplitude.com/hubspot/chart/f3tj7ie/edit/t2izuq7) (Amplitude) | [Significance calculation](https://tools.hubteam.com/significance/calculator?confidenceThreshold=0.05&vC-0=305&vE-0=11715&vC-1=469&vE-1=11715&vC-2=496&vE-2=11715&vN-0=Control&vN-1=Variation+A&vN-2=Variation+B)

| Metric | Control | Variants A & B (Predicted) |
| :---- | :---- | :---- |
| ESG (first page) Pageviews | 11,715 (33.3…% exposure) | 11,715 (33.3…% exposure ×2\) |
| Signups Initiated | 305 | 469 (×2\) |
| **Signups Initiated CVR** | **2.6%** | **4.0%** |

## **Leads Generated (Guardrail)**

[Data Source](https://looker.hubspotcentral.net/looks/68001?toggle=dat,fil&qid=pgFlLacgdFGpYqoWRm0rHO) (Looker)

| Metric | Control | Variants A & B (Predicted) |
| :---- | :---- | :---- |
| ESG Pageviews | 11,715 (33.3…% exposure) | 11,715 (33.3…% exposure ×2\) |
| Leads Generated | 6,326 | 6,326 (×2\) |
| **Lead CVR** | **54%** | **54%** |

# Considerations

* Our existing microapps are a source for \~30% of leads. Following discussions with the Marketing Acquisition team, we have agreed that any experiments for freemium acquisition must not affect existing lead generation. In order to protect the performance of our existing apps, we will:  
  1) Limit experimental signup motions to only occur **after lead conversion** in the app.  
  2) Only productize experiments where the results do not negatively impact the gross lead volume, even if significant gains are made in signups—i.e. An experiment where MRR from increased signups netted out lost MRR from decreased leads would not be considered for productization.  
  3) Test signup optimizations as experiments on a small subset of users in order to assess impact on the broader app before productizing results.  
  4) Include performance testing to ensure tools for user segmentation do not negatively impact page load performance.  
  5) Monitor (and build/maintain where needed) alerts for changes to the apps’ core performance metrics to the same standards we follow for HubSpot’s main app.

# Follow-up

* 

