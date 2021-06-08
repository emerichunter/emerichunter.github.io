---
published: false
---
## Optimizing Functions 2

In this post, I'm going to tell you about optimizing functions (hence the title). I am going to give you some tips and tricks to keep the performance at the maximum. 

My main first point will be to explain some options from the CREATE FUNCTION and then some other approaches to get the best out of your functions (this applies to procedures as well)

https://www.postgresql.org/docs/12/xfunc-optimization.html

From this very short entry within the official documentation. There are some takeaways already. The first thing to know is the function type you are using. 

### Volatile is stealing your speed


It might not seem obvious to all the users, but the default option for type is VOLATILE. The reason for this is very practical


Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
