---
layout: page
title: Portrait Studio Pro
description: Behind the scenes of my full stack ML application.
img: assets/img/projects/portrait_studio_pro/realtor.png
importance: 1
category: fun
related_publications: false
---

<image/>

## Introduction

If you’re chasing a polished LinkedIn photo but can’t spare an afternoon at a studio, AI headshots feel almost magical. The premise is straightforward: fine-tune a diffusion model on a small batch of your selfies, then let the network re-imagine you under flattering lights, crisp backdrops, and perfectly-pressed suits. With [Portrait Studio Pro](https://www.proaiheadshot.com), I turned that idea into a full-fledged web application that delivers more than 120 portraits in under two hours.

My own journey to that point was hardly instant. I cycled through half-a-dozen model architectures, hosting providers, and user flows before the pieces finally clicked. Along the way, competitors like Danny Postma’s HeadshotPro, Levels.io’s PhotoAI, and venture-backed Aragorn AI appeared on the radar. Far from discouraging me, their presence signaled genuine demand—“friendly competition” that forced better engineering decisions week after week.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        <div class="w-50 mx-auto">
            {% include figure.liquid loading="eager" path="assets/img/projects/portrait_studio_pro/hero_pic.webp" title="example image" class="img-fluid rounded z-depth-1" %}
        </div>
    </div>
</div>
<div class="caption">
    Examples of headshots generated from the app (first version)
</div>

## Building the Stack

I started by building out front end. I prototyped a layout in Figma and then coded it up with Next.js. I chose the App Router version of Next.js, which renders pages on the server by default. Client-side interactivity is opt-in: at the top of a file you add `use client` string to make React hooks like `useState` behave as expected. It felt a little quirky at first, but I adapted. At the time, the App Router version was brand new, so AI models were very unhelpful, having a tendency to use Page Router schema. The first version of the frontend took a couple of evenings to flesh out.

The backend is entirely Firebase. User profiles and order metadata sit in Cloud Firestore, while raw selfies and generated headshots rest in Cloud Storage. A fleet of TypeScript Cloud Functions orchestrates the rest: Stripe checkout, webhook processing, email reminders, and the heavy-duty job queue that triggers model training and inference. SendGrid handles the mailouts.

The machine-learning pipeline was the most comfortable piece for me to build. At a high level, every uploaded image passes through a pipeline: face detection, rotation correction, gender detection, brightness and blur checks, and a light histogram adjustment to even out exposure. Once an image set clears quality control, I do a LoRA fine-tuning of a flow-based generative model. Inspired by the DreamBooth recipe, the model learns a new token e.g. “photo of ohwx man”while staying anchored by generic regularization prompts and images such as “photo of a man.” 

To tune hyperparameters of the model, I build a dataset of people including my own face, my wife’s, obliging friends, even a few public-domain celebrities. Early models used convolution-based architectures based on Stable Diffusion. I found that LoRA of these models led to blurrier and smoother photos than full fine tuning, so these models did a full-parameter fine tune. The hyperparameters were a bit challenging, because it was easy for the model to collapse under all but very modest learning rates. Later models, with a transformer-based architecture, were fine with LoRa and produced better images. After running experiments, I found a LoRA rank, learning-rate schedule, and number of training steps that produced reasonably consistent result across the dataset. I didn't use any specific metrics, I just ranked outputs by my personal preference.

For Stable-Diffusion models, prompt engineering was really important, for example, positive prompts read:

```
professional headshot of ohwx man, standing in a modern office lobby, wearing a black suit and white shirt, 4K UHD, bokeh lighting, high detail, sharp focus
```
and negative prompts:

```
lowres, bad anatomy, bad hands, signature, watermarks, ugly, imperfect eyes, skewed eyes, unnatural face, unnatural body, error, extra limb, missing limbs
```

On the other hand, Transformer based models worked really well without extensive prompt engineering

To serve the inference pipeline, I started by using google cloud batch, but quickly ran into a lot of issues. For example, my GPU quota was set to 1 and I couldn't get approval for more than one concurrent GPU, so jobs backed up in the queue and were timing out. The retry mechanism then introduced some bug I never figured out; it provisioned disk-usage quota on each retry (even before the vm instance launched) and I ran into a Zone Resources Exhausted problem (due to the growing disk quota that was never really used). I gave up trying to figure this out and switched to containerized model for deployment on Replicate via the Cog toolkit. Replicate offers serverless GPUs inference, and cog made it fairly straightforward. I did have to crack open the generated Dockerfile to manually tweak some `c` libraries, but once that hurdle was cleared, everything flowed with almost no friction.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/portrait_studio_pro/downtown.jpeg" title="Downtown Style" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/portrait_studio_pro/blue.jpeg" title="Blue Style" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/portrait_studio_pro/woods.jpeg" title="Woods Style" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Examples of various styles available to select from.
</div>

## Launching

I learned quickly just how hard and time-consuming marketing is, so I kept this part short. For SEO, I did basic keyword research and created handful of geo-targeted landing pages ( e.g. “Professional Headshots Houston” for several locales in the US and Canada). After some time this led to some organic trickle. A brief Google Ads experiment converted at roughly three percent. This was solid until cost-per-click ballooned as the niche heated up. Surprisingly, the best traffic came from a paid listing on an AI-tools directory, where visitors arrived already “bottom-of-funnel.”

Google Analytics tracked every step from upload to checkout, while a modest email drip coaxed fence-sitters back with escalating discounts. Those three touches—after one hour, one day, and two days—turned enough “maybes” into paying customers.

## Lessons Learned
I learned alot about frontend development with React, Next.js and tailwind.css. I learned a fair bit about cloud computing. Firebase proved an ideal launch-pad: no servers to babysit, no manual TLS, no schema migrations—just ship. Over the long haul, the convenience tax may push core services onto cheaper or faster rails, but for version one it was perfect.

On the ML side, I learned about DreamBooth-style fine-tuning and LoRA. These are useful when data is scarce. A dozen well-lit selfies are genuinely enough to anchor identity, provided you guard your learning rate. A lightweight five-star ranking UI trumped any attempt at automated metrics; sometimes “vibe-checking” really is the best-available science.


## Closing Thoughts

From a handful of smartphone selfies to a gallery of studio-quality portraits, [Portrait Studio Pro](https://www.proaiheadshot.com) blends fast ML with no-nonsense product design. The stack will evolve—edge inference, instant previews, maybe even short video captures—but the current system already delights customers and pays its way.

If you’re eyeing a similar project, focus on rock-solid preprocessing, treat prompt engineering as a first-class citizen, and never underestimate the power of a single well-timed email. The technology is cutting-edge; the fundamentals of good user experience are as old as photography itself.

