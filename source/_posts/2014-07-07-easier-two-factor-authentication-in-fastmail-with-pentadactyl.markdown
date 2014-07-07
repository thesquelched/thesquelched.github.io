---
layout: post
title: "Easier Two-Factor Authentication in FastMail with Pentadactyl"
date: 2014-07-07 11:41:59 -0500
comments: true
categories: fastmail, pentadactyl, javascript
---

I switched to [FastMail](https://www.fastmail.fm) last month, and so far it's pretty
great (aside from my bayes spam filter not quite working optimally yet).
One of the selling points was that FastMail supports multiple two-factor
authentication providers, including Google Authenticator, which is a must
security-conscious folk like myself.  However, the login process is somewhat
annoying.  The login page looks like this:

<img src="http://i.imgur.com/3x7mUT6.png" alt="FastMail Login Screen" />

In order to use two-factor authentication, you need to click the "More" link,
which reveals the two-factor auth code input.  This feels unnecessary, and it's
particularly annoying to me, a [Pentadactyl](http://5digits.org/pentadactyl/)
user, as I have to either grab the mouse and click on the link, or I have to
exit insert mode, search for the "More" link, hit enter, and then re-enter
insert mode to input the two-factor code.  Also, unlike Gmail, there's no
option to allow an authenticated computer to skip the two-factor auth step in
the future.

FastMail [openly admits that the process is a pain](https://www.fastmail.fm/help/account/2fa.html),
and that they intend to streamline it at some point in the future, but damnit,
I demand satisfaction *now*.  While there's nothing I can do about the latter
issue, I sure as hell can take care of the former.  Pentadactyl allows you
[create custom Javascript commands](http://5digits.org/help/map.xhtml#:command),
which you can run automatically using
[autocommands](http://5digits.org/help/autocommands.xhtml).  Using these two
features, I should be able to write a command to locate and click the "More"
link (which is actually a `<span>` element), which I can then call
automatically on page load with an autocommand.

Let's try the simplest possible approach: get the DOM element corresponding to the link, then call its `click` method.  This goes in your `$HOME/.pentadactylrc` file:

```
command! fastmail-twofactor -description "Show FastMail two-factor auth window" -javascript
    \ content.document.querySelector('div.content #problem span').click();

autocmd DOMLoad www.fastmail.fm :fastmail-twofactor
```

Note that, in order to access `document`, which would ordinarily be a global
variable, you need to use the `content` object provided by Pentadactyl.  You
can then reload the configuration using the `:rehash` or `:reh` command.
However, upon reloading the page, the dialog is still collapsed.  Manually
running the `:fastmail-twofactor` command works, though, so there must be a
problem with the autocommand.  The `DOMLoad` event fires when the page loads,
so the command should work as long as the "More" link is present on load;  one
way this could fail is if some content is added to the page asynchronously
after the DOM loads.  You can confirm this using the Firefox debugger.  It
turns out that `homepage.js` is loaded on the page; setting a breakpoint at the
beginning and stepping through the execution reveals that the page loads
without the login window, one to load and display it shortly afterward.

Unfortunately, there's no Pentadactyl event for loading individual objects on
the page, so we'll have to look for a pure Javascript solution.  DOM3, which has been supported by major browsers for a while now, provides the
[MutationObserver object](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver),
which allows you to observe changes in the DOM.  We should be able to use this
to detect when the "More" link is added, after which we can trigger the click
event.  However, just to be safe, we should check if the link already exists
beforehand.


```
command! fastmail-twofactor -description "Show FastMail two-factor auth window" -javascript
    \ var problem = content.document.querySelector('div.content #problem');
    \ 
    \ var more_link = problem.querySelector('span');
    \ if (more_link != null) {
    \   more_link.click();
    \   return;
    \ }
    \
    \ var obs = new MutationObserver(function(mutations) {
    \   mutations.forEach(function(mutation) {
    \     for (var i = 0; i < mutation.addedNodes.length; i++) {
    \       var node = mutation.addedNodes.item(i);
    \       if (node.tagName.toLowerCase() === 'span') {
    \         node.click();
    \         obs.disconnect();
    \       }
    \     };
    \   });
    \ });
    \
    \ obs.observe(problem, {childList: true});

autocmd DOMLoad www.fastmail.fm :fastmail-twofactor
```

Voila!  The login window now automatically expands when you enter the page.
