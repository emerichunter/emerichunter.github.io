---
published: false
---
## Plotting sadf

Let's talk about sysstat... Well no, let's talk about Postgres first...

Actually let's talk about sysstat for Postgres. And about graphs. Dumb graphs. 

### And there was light

The sysstat package, which can be found on many linux based servers nowadays is very usefull. 
It can collect data about what is happening to your CPU, RAM...

I won't get into specifics today, this is not the point. I would like to present it from the database perspective. It's all about persperctive. The Postgres user can access sysstat informations. More precisely everything collected by `sar`. 

Sar can help you know what is going on right now and what happened in the past, since it collects information for each day of the month. It's a pretty big tool and a database administrator who is not acustomed with sysstat can very easily get confused. So first let's use something much easier to grasp :  sadf. Sadf work the same excepts it only gives last 24H of activity and can be very exported natively in CSV files. 

### CSV you say ? 

Yes, you guessed it! You can graph that very easily too.

Imagine, you have no GUI, no graphs, no monitoring... Well, sometimes that's my case. 
Imagine, you cannot get any money to pay for the expensive fancy tools... (bosses and all)

How can you graph things easily ? Well GNUplot can do just that for you, for free. 

But you need to get everything to work, and not spend too much time on this endeavor. 

So let's get a tool to do just that... Wait! I already made one! 


### Enters the Savior

Some time ago, I made this tool and today I will just blog briefly about it. Because sometimes, this kind of things can help. Depending on the context. 
If you are poor, you have nothing
