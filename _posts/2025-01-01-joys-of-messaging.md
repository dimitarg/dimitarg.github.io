---
title: "The joy of messaging without a message bus"
date: 2023-12-13T20:00:00Z
published: true
categories:
  - Functional Programming
  - System design
tags:
  - fp
  - postgres
  - messaging
---

# Summary

Depending on the usecase, it's possible to build a production-grade, near-realtime messaging system on top of just PostgreSQL, foregoing messaging middlewares such as Kafka entirely. This will take a certain amount of care, so while it can be simple it might not be that easy. After all, you'd be venturing off the beaten path.

The cost of Kafka and similar software, in terms of operation, complexity and financial overhead is non-negligible, which is what prompted us to look for an alternative.

We propose an architecture and sample implementation (in Scala) which might be a good fit if

- You haven't already paid for and invested in a messaging bus
- You already run PostgreSQL
- Your throughput requirements are far from Google scale - which, statistically speaking, they are.

If the above description does not fit your usecase, you might still find this article a useful exploration in system design. 

# Introduction

I've been out of a contract for a while - tough market, I know - so I was looking for a personal project to brush up on my programming and system design skills, and get idustry-grade experience with Scala 3 as well.

My initial pick was an OAuth / OpenID Connect server implementation, but after faffing about with documentation for a week or so, I started to drown in the 20-ish RFCs involved and realised this would take me around three years, bankrupt me completely and I'd probably lose my ability to communicate with humans in the meantime. For the sake of my financial stability and mental health, I needed a less ambitious goal.

Had I continued on the path of building an authentication and authorisation provider, one of the next pieces needed would have been a communications service - at the very minimum, something capable of sending emails - for tokens, password resets, security notifications and such. It's an extremely common piece of infrastructure - I'd have to think really hard to come up with a system I've worked on in the past ten years that hasn't had to send email to people.

In a comms service, the actual email sending is performed by some sort of SMTP-capable server. It's an interesting topic in itself, since hosting an SMTP server in 2025 that's actually able to deliver email to people and not get rejected or flagged as spam is a surprisingly complicated endeavour. So much so that we usually default to using a managed service, such as Amazon SES, SendGrid, Mailgun, etc. They do come at a non-negligible cost, which goes to tell you you'll probably have a hard time building your own.

There's a certain amount of infrastructure surrounding the actual physical email sending, and that's what we'll focus on in this light but lengthy read.

## Durability

In a typical comms service, the most important goal is durability. We cannot just shoot out emails into the ether and hope for the best, because that's just sloppy. We need to record the fact. When a request is made to send an emails, there should be two eventual outcomes:

- The email does get sent, and that fact gets recorded in our system, for audit purposes and otherwise;
- The attempt to send results in an error, and that fact gets recorded, for automated or manual retrial, audit purposes and others.

Specifically, there should be no outcome where our system accepts a request to send a message, and that fact is forgotten - due to external system error, hardware fault or service shutdown in the midst of processing, or any other reason.

## The case for messaging middleware

Not losing messages is our top priority. Second on the list is near-realtime processing - the recepients should receive their emails shortly after the message sending was requested, for some definition of 'shortly'. Specifically, this means that we can't just use batch processing - i.e. firing up a task that reads unsent messages from the database every five minutes and sends them in bulk is not adequate.

> Note these durability and near-realtime requirements are crucial for emails where we send a specific message to a specific (set of) recepient(s), and where not being able to deliver the email is deemed a major issue. These are often dubbed "transactional emails". As an example, think "Order confirmation", "Payment receipt", "Invoice", "Password reset". A counter-example would be a generic email to many recepients as part of marketing campaign. In that example, depending on the usecase, both the durability and latency requirements might be relaxed. We're only discussing transactional emails in this article. 

To sum up, our usecase demands a system giving us the capability to produce and consume messages in a durable and near-realtime manner. 

A standard and tried approach to implement this usecase is to use a combination of a relational database and a message-oriented middleware such as Kafka, AWS Kinesis, RabbitMQ or such. Roughly, the scheme is

- Upon receiving a request to send out a set of messages, record those in the database, with a status of "Scheduled";  publish corresponding messages to your message broker of choice; and return to the client that the request was "Accepted";
- A consumer of that message topic / queue takes messages, attempts to send them to the SMPT-capable server of choice, and upon success
  - Updates their status in the database to "Processed"
  - Acknowledges to the message broker the message was processed (in the case of Kafka, by committing the message's offset)
- Upon error, automatic retrial can occur, but eventually we need to give up, record the message in the database with status "Errored", acknowledge it in the message broker and move on to the next one. These errored messages can then be actioned automatically or manually, either via sourcing them from the database state, or sending them to a "Dead letter queue".

This approach is so standard that I would personally default to it when tasked with building an email comms service. It works, it provides at-least-one delivery semantics (which is the best we can do, since deduplication in general cannot be achieved with SMTP), and via the message broker it provides unbounded horizontal scalability - until the RDBMS or the SMTP service becomes the bottleneck.

## The cost of messaging middleware

Kafka and friends are great, but they come at a a cost. If your organisation is already utilising message middleware, the costs are already being paid, but if you're only introducing it now, you need to acknowledge these.

There's organisational cost: once you introduce message middleware into your architecture, you need everyone on the squad to be familiar with the technology, and at least a few people proficient in it. 

There's complexity cost, as you're introducing an additional distributed system in your stack. This introduces new programming models, new libraries, new failure modes of the system, and new concerns in infrastructure provisioning and operation.

There's the direct financial cost. At the time of writing, an entry-level 3-node AWS MSK cluster comes out at circa $600 per month, sans tax. For a large enterprise this amounts to an accounting error, but for a frugal startup it is substantial, especially in the early stages of trying to make a service financially viable.

Or maybe you're one of the few businesses that already owns their hardware, you've over-provisioned and have the spare resources, and you're willing to operate Kafka on your own, paying the operational cost. This might be a good fit if you already have the expertise. 

If not - among other things, you're now in the business of operating a distrubuted consensus algorithm in production. Congratulations! Just a reminder that your three-node cluster (or six, if you're running Raft on dedicated nodes) that's running fine is one node failure and one network partition away from complete system outage. All this to say, it's not an endeavour to be taken lightly.