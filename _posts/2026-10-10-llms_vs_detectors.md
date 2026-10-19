---
layout: post
cover: 'assets/images/cover-navy-abstract.jpg'
navigation: True
title: Child detection in multimedia - can we replace the classical detectors with multimodal large language models?
date: 2024-04-23 12:00:00
tags: experiments
subclass: 'post tag-experiments'
author: weronika
categories: weronika
---

Nowadays, multimodal large language models (MLLMs) seem to be the answer to almost every computer vision task. However, they are not usually used as a replacement for classical object detectors such as YOLO. Replacing fast and reliable methods with much slower and more resource-intensive MLLMs rarely makes sense. The situation is different for tasks that fall outside the scope of off-the-shelf detectors, such as detecting objects that are not included in the COCO categories. Collecting additional data to train or fine-tune models can also be time-consuming, making MLLMs an interesting alternative for these kinds of problems.

Child detection is one example of such a task. High-performing person detectors are readily available, but distinguishing children from adults is a different challenge. Combining person detection with an age estimation method to filter out adults without fine-tuning both methods can lead to disappointing results. 
<!--more-->
In this experiment, two fundamentally different approaches to child detection were compared: a classical object detector (YOLO26X) and an MLLM (Qwen3.5-27B). After some preliminary experiments, a third approach was added: a hybrid solution in which the person detector handles localization while the MLLM estimates age. This approach seems to combine some of the strengths of both architectures.



|                |                                                                                  Supplementary materials:                                                                                   | 
|:---------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:| 
| Link to report | ["Classical Object Detectors vs. MLLMs for Child Detection in Images: A Preliminary Study"](https://github.com/weronikagutfeter/MinorsReport/blob/main/reports/2026_llms_and_detectors.pdf) |
| Link to codes  |                                 [experiment: llms_and_detectors](https://github.com/weronikagutfeter/MinorsReport/tree/main/experiments/llms_and_detectors)                                 | 

### The Parenting Slop Dataset

The story behind this experiment started when I was on maternity leave. I noticed that social platforms started suggesting more movies with children to me (which wasn't surprising). Among them, I got a lot of synthetically generated movies of different animals stealing or attacking babies. The more I clicked on them, the stranger the animals appeared in the clips. My top find was a cassowary escaping with a little kid in its beak (I even needed to check the bird's name, because it's not popular in Poland, where I live). At some point, I started downloading some of the clips, since I thought other people wouldn't believe how strange the content was. As a researcher, and because my last project involved working on NSFW content with children, I decided to add more diverse data to synthetic movies and to create a database for further analysis of children's presence in real and AI-generated videos. However, I’m aware of the bias in the dataset toward clips with animals and children.

![image](assets/images/posts/Samples_gt.jpg)

I called it the Parenting Slop Dataset and made it available online on the platform Zenodo ([Parenting Slop Dataset on Zenodo](https://zenodo.org/records/22092731))

The collection consists of 20 movie clips showing children in various situations. It contains both real and synthetic data. Selected videos show subjects in close-up and from a distance; children may be in groups, occluded, moving, or not looking at the camera, making the data more challenging. All the videos were trimmed to 10 seconds to have a uniform length. They were then sampled at 5 fps and saved as JPG images, resulting in 50 frames per video, for a total of 1000 frames. Keyframes were annotated with rectangular bounding boxes for the **minor** category using LabelMe. The minor category is defined as a child with an apparent age below fourteen years. Adults and other objects in the images are not labeled. 

### Experiment


The main goal of this experiment is to identify all children appearing in the images from the Parenting Slop Dataset.
A key assumption is to minimize or eliminate the need for additional data collection for training and to avoid cloud-based infrastructure. Therefore, all models considered in the comparison are open-source and open-weighted.

The following no-finetune approaches are explored:

1. **Classical person detector + age estimation:** A conventional object detector is first used to identify all people in the image. An age estimation model is then applied to the detected regions to distinguish children from adults and filter out the latter.
1. **MLLM-based child localization:** An MLLM is prompted to identify and localize all children visible in the image, providing a direct end-to-end approach to child detection.
1. **Hybrid solution** Combining the classic detector and age estimation MLLMs


### Classical person detectors

State-of-the-art (SOTA) object detectors are commonly trained and evaluated on COCO, which defines a fixed set of object categories, including person. Although this category does not distinguish between adults and children, it can provide a basis for child detection. A straightforward way to adapt a general-purpose person detector to the task of child detection is to first localize all people in an image and then determine their age. 

### MLLM-based child localization

For the detection task, two types of prompts were tested:

* all-people prompt:

> Detect all the persons in the image and return their locations and descriptions in the form of coordinates. Results should be in JSON format: { "user": {"bbox": [x1, y1, x2, y2], "description": "description of the person", "label":"adult/child" } }

* and children-only prompt:

> Detect all the children in the image and return their locations and descriptions in the form of coordinates. Results should be in JSON format: { 'user': {'bbox': [x1, y1, x2, y2], 'description': 'description of the person', 'label':'adult/child' } }.


Despite the detailed instructions, the models did not always return valid JSON. The outputs generated by the MLLMs required post-processing before they could be used for evaluation.

### MMLM-based age estimation

Classical object detectors are fast and effective at detecting people, but age estimation can become a bottleneck, as incorrect predictions may cause actual children to be filtered out. MLLMs, on the other hand, can detect children directly and provide additional context through descriptions of the people in an image. However, even the smallest models took about 100 times longer to run than the baseline person detector, and their outputs required rule-based post-processing to handle inconsistencies in the response format. To combine the strengths of both approaches, an additional experiment was conducted using a hybrid solution in which YOLO26X detected people and an MLLM classified each detected person as a minor or an adult. 
The prompt to the MLLM was following:
> What age is the person in the image? The result should be either adult (for a person older than 14) or minor (for a person younger than 14). If there is no person in the image, return unknown.

#### Overview of results

The detection results were evaluated using the standard COCO detection metrics: AP (Mean Average Precision), AP@0.5 (Average Precision at IoU=0.50, AP@0.75 (Average Precision at IoU=0.75 and AR (Average Recall with a maximum of 100 detections per image).


| Approach                                            | Details                                   | AP | AP@0.5  | AP@0.75 | AR | 
|-----------------------------------------------------|-------------------------------------------|----|---------|-----|----|
| Baseline person detector                            | YOLO26x + no age estimation               | 0.546| 0.672   |0.599|**0.772**|
| Detector + Age estimation with classification model | YOLO26x + InsightFace                     | 0.223| 0.263   |0.245|0.302|
| Detector + Age estimation with classification model | YOLO26x + MiVOLO                          | 0.428| 0.500   |0.451|0.454|
| End-to-end MLLM                                     | Qwen3.5-4B _(all-people prompt)_          | 0.401| 0.726   |0.399|0.533|
| End-to-end MLLM                                     | Qwen3.5-4B _(children-only prompt)_       | 0.417| 	0.733  |	0.413|	0.567|
| End-to-end MLLM                                     | Qwen3.6-27B   _(all-people prompt)_       |0.618	| 0.822   |0.693	| 0.701 |
| End-to-end MLLM                                     | Qwen3.6-27B   _(children-only prompt)_    |0.669	| **0.897**	  | **0.758**	| **0.772**|
| End-to-end MLLM                                     | Qwen3.6-35B-A3B  _(all-people prompt)_    | 0.619| 0.828   |0.705|0.756|
| End-to-end MLLM                                     | Qwen3.6-35B-A3B  _(children-only prompt)_ | 0.585 | 	0.829	 | 0.645	| 0.734|
| Detector + Age estimation with MLLM                 | YOLO26x + Qwen3.5-4B        | 0.675| 0.809 |0.734|0.752|
| Detector + Age estimation with MLLM                 | YOLO26x + Qwen3.6-27B       |**0.682**| 0.810 | 0.743 | 0.734     |
| Detector + Age estimation with MLLM                 | YOLO26x + Qwen3.6-35B-A3B   | 0.593| 0.701 |0.638|0.628|

[//]: # (### Conclusions)