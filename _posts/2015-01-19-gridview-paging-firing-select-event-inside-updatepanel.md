---
layout: post
title: GridView paging firing select event inside UpdatePanel
---

I am working on an application containing a few [UpdatePanel](http://msdn.microsoft.com/en-us/library/system.web.ui.updatepanel%28v=vs.110%29.aspx) elements to give the appearance of running like a single-page application. For the most part, this works, but I ran into an issue where paging through a GridView in an UpdatePanel would call the Select Command of the GridView every time the page changed. I knew that this was not happening on other pages built similarly, so I looked up the difference and realized that the CommandName needed to be specified to prevent the intended Select action from being called every time.

{% highlight c# %}
protected void GridView_OnRowCommand(object sender, GridViewCommandEventArgs e)
{
if (e.CommandName.Equals("Select"))
{
//do something
}
}
{% endhighlight %}