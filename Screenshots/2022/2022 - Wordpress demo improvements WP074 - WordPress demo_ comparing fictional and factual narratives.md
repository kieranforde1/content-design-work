# WP074 \- WordPress demo: comparing fictional and factual narratives

[Kieran Forde](mailto:kiforde@hubspot.com)  
**Experiment dates**: 16/11/2022 \- 3/1/2023

## **HYPOTHESIS:** 

We believe that if we update the content of the WordPress demo to remove fictional parts of the narrative \- such as “Imagine you work at SpotFit gym” and “Alex is now a new lead“ \- the number of people who complete the demo will not drop.

**CONTEXT**  
We know that users who complete the demo are more likely to go on to sign up for HubSpot.  
   
Right now, the WordPress demo is not showcasing the most valuable features to users. We have designed improvements to the demo, but we want to roll out our improvements incrementally, rather than rolling out many changes at once. This will help us understand which changes are most impactful. 

This experiment is the first step. If using a factual narrative makes no difference to completion rates, we can more easily experiment with gradual changes to the demo in future. If a fictional narrative performs better than a factual narrative, we may need to drop the incremental approach and roll out many changes at once (making once change to a fictional narrative can have knock-on effects on many steps, so will require a different approach).

WP065, testing adding a preview button to the plugin signup page, showed a demo [completion rate of 31.4%](https://analytics.amplitude.com/hubspot/chart/ib1kh3r).

It’s also worth checking out [this chart](https://analytics.amplitude.com/hubspot/chart/sjczwrh) that shows how much CVR is impacted from the preview demo CTA on the signup page.

## **EXPERIMENT DESIGN:** 

| Testing location | HubSpot for WordPress plugin \- a demo link appears on the signup page and after signup completion |
| :---- | :---- |
| **Audience** | All users signing up for the plugin, and existing users that chose to see the demo again |
| **User Activity**  | Progress through each step of the demo, and total demo completion rate [Chart (to be updated)](https://analytics.amplitude.com/hubspot/chart/jsi1m3d) |
| **Control Copy** | See table below |
| **Variant 1 Copy** | See table below |
| **Measure** | Demo drop off Completion rate of the demo |
| **North star metric** | Signups |

## **Copy changes**

[Link to the demo on QA](https://app.hubspotqa.com/wordpress-plugin-ui/demo)

| Step | Control copy Fictional narrative | Variant copy Factual narrative | Complete in CE tool |
| :---- | :---- | :---- | :---- |
| Intro | **Here's a quick demo** Imagine you work at SpotFit gym. You've just signed up for the HubSpot CRM to help you grow your online membership list. \[Start demo\] | **Here's a quick demo** Let’s look at how you can use the HubSpot for WordPress plugin to help you grow your business. \[Start demo\] |  |
| 1 Dashboard | It's the beginning of the quarter. The numbers on the dashboard tell you that SpotFit needs some good quality leads to work on. \[Next: Capturing leads\] | Your dashboard tells you what’s happening in your business. For example, this chart shows how many deals you’ve closed compared to your target. \[Next: Capturing leads\] |  |
| 2 Blank browser window background | With this plugin, you can capture a visitor's information with ease. No help from a developer needed. \[Next step\] | With this plugin, you can capture a visitor's information with ease. No help from a developer needed. \[Next step\] |  |
| 3 Forms, overlaid on blank browser window background  | Next you add a special offers form to your homepage. Alex is filling it out now. When she hits subscribe, her details will be added straight into your HubSpot CRM. And you can send her an email back automatically too. \[Next: Contacts\] Which plans are you interested in? \[Corporate Online\] | This form lets your customers sign up for special offers from your homepage. When they hit subscribe, their details will be added straight into your HubSpot CRM. You can set up an automatic email reply too. \[Next: Contacts\] Which emails are you interested in? \[Special Offers\] |  |
| 4 Contact page | Alex is now a new lead in your contacts database, and this is her contact record \- it’s got everything you know about her so far. \[Next: Tracking activity\] | Here’s your new lead in your contacts database. This is their contact record \- it’s got everything you know about them so far. \[Next: Tracking activity\] |  |
| 5 Contact page | You see Alex has visited your site again since she filled the form. She's definitely interested in what you have to offer. \[Next: Lists\] | You can see a lead’s activity on your website, including when they filled in the form. This helps you to see who’s interested in what you have to offer. \[Next: Lists\] |  |
| 6Lists: table of all lists | Because Alex said she was interested in online corporate plans, she was added to your corporate online offers campaign list. Learn more about lists in HubSpot. **Next: Click the 'Corporate Online Offers' list name.** | Use lists to organize your contacts. This list automatically shows you everyone who filled out the Special Offers form we looked at earlier. Learn more about lists in HubSpot. **Next: Click the 'Special Offers' list name.** (change the name of the item in the list to Special Offers too) |  |
| 7Lists: detail page | Now you have a list of everyone interested in your corporate online offers, let’s engage them with an effective email.\[Next: Email campaigns\] | Now that you have a list of everyone interested in your special offers, let’s engage them with an effective email. \[Next: Email campaigns\] |  |
| 8 Email editor | Your email may be going out to thousands. But with HubSpot, you can quickly add in a personal touch using personalization tokens. What are personalization tokens? \[Next: Customizing emails\] | You can easily use the information your leads share with you to add a personal touch to your emails. What are personalization tokens? \[Next: Customizing emails\] |  |
| 9 Email editor: Highlight “button” in drag and drop editor | Instead of building your special offers email from scratch, you customize a ready-made template. It's easy and fast, and you don't need help from a designer or developer. Let's try it. **Next: Drag and drop this button onto the template. Email copy** Thank you for signing up for our special offers. Each month you’ll be the first to hear about our huge corporate discounts on online membership, online classes, and class packages. To welcome you aboard, we’d like to offer your employees a free online class.  **Button text**\[Claim your class\]  | Ready-made templates make it easy and fast for you to create targeted emails to send your leads. You don't need a designer or developer to make it look great. Let's try it. **Next: Drag and drop this button onto the template. Email copy** Thank you for signing up for our special offers. Each month you’ll be the first to hear about our discounts. To welcome you aboard, we’d like to offer you 20% off.  **Button text**\[Claim 20% off\]  |  |
| 10 Email editor: Highlight “Review and send” button | Your special offers email is looking good, so it's time to send it to your list. (Not for real \- you're still in a demo). **Next: Click 'Review and send'.** | Your special offers email is looking good, so it's time to send it to your list. (Not for real \- you're still in a demo). **Next: Click 'Review and send'.** |  |
| 11 Email report  | As soon as you send your email, you start seeing delivery rates and engagement straight away. \[Next: Top engaged contacts\] | As soon as you send your email, you can start seeing delivery rates and engagement straight away. \[Next: Top engaged contacts\] |  |
| 12 Email report: Amervis email address highlighted | 🎉🎉 Alex has already clicked the links in your email 4 times. You've turned her from a visitor into a good lead, likely to become a customer. Best of all, you did it using tools that are 100% free, forever. \[End demo\] | 🎉🎉 Your report shows you your most  engaged contacts. You can see how many times your leads have clicked links in your email.  With HubSpot, you’ve turned website visitors into good leads that are likely to become customers. Best of all, you can do all of this using HubSpot’s free tools\! \[End demo\] |  |

## **Duration:**

15 days to reach significance, data below:

| Link to [Estimation](https://tools.hubteam.com/significance/duration-estimator/?branchCount=2&confidenceThreshold=0.05&controlCVR=0.26&cvrIncrease=0.03&dailyVisitors=430) | Control | Variant | Diff |
| :---- | :---- | :---- | :---- |
| Exposures | 6400 | 6400 | 0 |
| Demo completion rate | [32.3%](https://analytics.amplitude.com/hubspot/chart/9m2ou9o) | 35.3% | \+3% |

## **RESULTS:** 

[**Link to Amplitude dash**](https://analytics.amplitude.com/hubspot/dashboard/4mxxi7c)

Demo completion is significant at 99% confidence with a power of 100%  
**![][image1]**

---

Sign up CVR increase is significant at 99% confidence with a Power of 99.7%.  
**![][image2]**

## **Key Takeaways \+ Next Steps:** 

**Key Takeaways:** 

* The variant underperformed on dropoff until the final step where it outperformed the control, so much so that it was overall a significant increase.  
* The variant gives us much more control over the demo improvements, as the steps aren’t tied to a narrative. This means we can remove/add/change steps as necessary  
* Those who complete the demo are [far more likely to sign up](https://analytics.amplitude.com/hubspot/chart/exn4zik)

**Next Steps:**

* Productize the variant  
* Iterate on screen order and overall demo length, and improve navigation of the demo  
* Run a final experiment with a story-led narrative in the new order, to verify whether factual has a higher completion rate
