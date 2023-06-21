---
layout: post
title:  "Acing Tech Interviews in 7 steps"
date:   2023-04-11
published: true
comments: true
categories: [Angular, Interviews, State management]
cover: assets/acing-tech-interviews-in-7-steps.jpg
description: "Use these simple steps to ace any tech interview."

---

A few weeks ago we did a Twitter space together with [Chau Tran](https://twitter.com/Nartc1410){:target="_blank"}. It was hosted by 
[Daniel Glejzner](https://twitter.com/DanielGlejzner){:target="_blank"} and 
[Pawel Kubiak](https://twitter.com/pawelkubiakdev){:target="_blank"}
We want to thank each of these people for what they are doing for the community. You guys are truly awesome!!
You can find the recording of that space here:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">🎉 Just hosted incredible <a href="https://twitter.com/hashtag/Angular?src=hash&amp;ref_src=twsrc%5Etfw">#Angular</a> Job Interview space!<br><br>Guests: GDE <a href="https://twitter.com/Nartc1410?ref_src=twsrc%5Etfw">@Nartc1410</a> and <a href="https://twitter.com/brechtbilliet?ref_src=twsrc%5Etfw">@brechtbilliet</a> - co-host <a href="https://twitter.com/pawelkubiakdev?ref_src=twsrc%5Etfw">@pawelkubiakdev</a>!<br><br>🥇 A gold mine of knowledge for those seeking Angular jobs or looking to hire engineers!<br><br>We covered in-depth Angular topics!<a href="https://t.co/wKH7d8VRNe">https://t.co/wKH7d8VRNe</a></p>&mdash; Daniel Glejzner (@DanielGlejzner) <a href="https://twitter.com/DanielGlejzner/status/1644309217296171008?ref_src=twsrc%5Etfw">April 7, 2023</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

In this article, we will learn a few steps to ace any tech interview. It doesn't matter if you are an Angular or React developer, or if you are focused on Java, .Net, etc...
We will go through 7 tips that will help you **ace any tech interview**. This focuses more on the frontend developer/architect role but all of these tips are valid for any tech interview.

## Tip 1: Get to know the company

Yes, we probably know this one but are we actually doing these steps?
- Did we look up the company online? Are there any investment rounds that we can find information on? Can we estimate the revenue, and amount of employers? Can we describe in 2 sentences what this company provides to its clients?
- We should ask ourselves why we want to work for this company. If we can convince our self then we can use that to convince our interviewer.
- Dig through the company its social media profiles and follow them. The information that we learn from that can show dedication, motivation and shows true interest from our side. Is there an interesting article that they wrote? We should read it and casually mention our positive opinion on the matter when we introduce ourselves and why we want to work for that company.
- Are there any names of the interviewers in the meeting invite? Look them up on Twitter and LinkedIn and learn as much as you can about them. There is a chance your interviewer has written an article that you can read and can tell them about. These are free bonus points because they might be humbled and it shows we care!
- Do we know what they expect from you? Are you a good fit? Don't be shy. No need for imposter syndrome here, but if we can convince ourselves upfront that we are a right fit for the job, we will radiate that vibe on the interview as well.

## Tip 2: Be happy

Congratulations!
You have earned that interview! That's awesome!! **Smile!!**. Be energized and be confident. After all... You have a meeting with a fellow geek that loves the same technology that you are so passionate about. It can be such an awesome experience to talk about the thing we love. Seriously... Smile! It doesn't hurt and it's a sign of confidence. If they believe that you believe you are the right fit. You are already halfway there!
Before an important meeting or solicitation interview I usually jump up 20 times with my eyes closed visualizing myself acing that interview (on remote interviews). It boosts my confidence level and I want to radiate energy and charisma. Don't get cocky or arrogant, but show your enthusiasm that you have been invited for that interview. Also, these people have to work with you, and showing that you are a pleasant person never hurts.

## Tip 3: Don't get stressed

Did you know that it's very hard to find tech talent? Did you know it's super hard to find people that are motivated? Taking interviews is one of the services we provide for our clients here at Simplified Courses. You can take our word on it that it's hard to find people with some motivation. If you are not motivated then this article might not help you, but if you are... I have good news for you.
**Being motivated is way more important than you memorizing textbook questions or documentation.**
Try this exercise: Let's close our eyes and imagine that we hope that we will ace this interview. How does that make us feel? Our brain becomes insecure and starts feeding itself the hypothesis that we will fail. Of course, it will try to do that, because otherwise we will be disappointed if you don't get the job. So it builds a wall to protect us and then if we do succeed in the interview, we can be extra happy. That's not a healthy mindset.

Now let's try this: Let's close our eyes and see ourselves acing that interview. **We just know** you are going to get this job. We just will, we can feel it in your bones. Because we are the right fit. How does that make us feel? First of all, the job looks way nicer than before. We also radiate confidence and feel happy instantly. 

Look, if we don't have the job we can be extra disappointed but maybe that disappointment will drive us even harder.
But the truth is, we have just increased our chances of acing that job a lot!

## Tip 4: Use the questions

Sometimes we get questions that we don't know how to answer. That's fine, we are no computers or walking search engines. **Be honest, but use the question!**

Don't say something like: "I don't know" or "I haven't thought about that actually". It creates an awkward silence. Take this question for instance: "What are the different Angular component lifecycle hooks and in what order are they executed". This stupid question could use this great answer: "I don't know about the order honestly, I know about `ngOnChanges()` but `ngOnDestroy()` is the most important one to avoid memory leaks.
**BOOM!** Now you can start talking about RxJS memory leaks or the fact that injectables provided in components can also implement the `ngOnDestroy()` lifecycle hook. You are giving the interviewer information he or she wants, even if you didn't know the answer completely.
You have not dodged the question, you have simply stated that you didn't know the order and shown that you know that injectables can also implement this. If you are passionate about Dependency Injection you can start talking about that.

So in short: Be honest! Use the questions! Your interviewer wants to learn as much as possible about you and your experience with the technology so use that time. Seriously, it's fun and it builds trust. 
Also, interviewers struggle with their questions as well. Their goal is not to validate if we can answer all their questions, they want to get as much information out of us as possible.

## Tip 5: Reverse the roles

We don't want to get caught in a firm where everything is written with old tech. We probably don't want to work with bad practices because the architect is biased or whatever.
We don't want to be cocky, but we want to know what their architecture looks like. We want to know which technologies the company uses, whether it's an agile environment or they love the good old waterfall principle. 

Let me give you this example:
Don't ask your interviewer how Change Detection in Angular works, but ask them what their opinion on the usage of the `OnPush` strategy is. If you really want to blow them away ask them how much of their code is run in the inner zone (which will trigger zone.js all the time)

## Tip 6: How do you stay up to date

It's a question I always ask the candidate, but if your interviewer doesn't ask it, mention it anyway. You could talk about a great conference you attended, a book you enjoyed or simply a blogger you follow. Even dropping some tweets that inspired you can do the job. This shows that you are taking some free time to stay up to date with a very vibrant technology field.
Open-source can also be a beautiful addition, although it is not mandatory.

The next thing you could talk about is training. Did you follow any training, online courses or some one-on-one coaching sessions. If you are completely self-thought, that's totally fine and that's something you want to mention as well. Be sure to tell the interviewer how you became self-thought. Try to add some real practice in there like pet projects that you take on as a hobby.

## Tip 7: Prove you are up to date

Proving you are up to date is a vastly different thing than showing how to stay up to date.
Let's say that you are going for an interview as an Angular developer. Make sure you know which is the latest version and which next version is going to be released soon. Maybe even learn when it will be released and what the new features will be.
In the Angular framework, there is this huge thing coming in Angular 16 which is called Signals. It would be sad if you couldn't give your 2 cents about that.

## Conclusion

It's a wrap! Don't be afraid of these interviews. They are awesome! They allow you to show how passionate you are. If you don't have the answer to a question, be honest and talk about what you do know about that topic. Realize that you are motivated, confident and one-of-a-kind. **You will ace this interview**

Special thanks to the reviewer [Daniel Glejzner](https://twitter.com/DanielGlejzner){:target="_blank"}

If you like to learn directly from me, check out my [Angular Training](https://www.simplified.courses/angular-training){:target="_blank"} and [Angular Coaching](https://www.simplified.courses/angular-coaching){:target="_blank"}