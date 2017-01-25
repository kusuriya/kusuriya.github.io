---
layout: default
title: PowerShell and the cryptic error message!
Date: 2017-01-25 01:30:00 +/-0000
published: yes
tags: 
    - Tools
    - PowerShell
    - DSC
---

# {{ Page.title }}

DSC is a great idea that is still in the process of being completely baked out in windows.
As such there are still a few quirks and issues, one of them being error messages.  
Today I ran into one of those ever great cryptic messages ``` Failed to get the action from server http://server/PSDSCPullServer.svc/Action(ConfigurationId='foo')/GetAction ```,
so lets talk about it.

<!-- more -->

After a bit of investigation this was caused because my server, while it could resolve the address for the DSC pull server, it could not actually reach it over the network.  
This could have been probably better worded as "Operation "getting action from server http://server/PSDSCPullServer.svc/Action(ConfigurationId='foo')/GetAction" timed out",
or if the error had an http status code attached to it "There was an error getting action from server http://server/PSDSCPullServer.svc/Action(ConfigurationId='foo')/GetAction the server responded: 403"

so in an effort to give some quick, albiet common sense, troubleshooting tips for this here we go:
- Check the connectivity from the server to the pull server, this should go without saying. Can you ping it? Can you trace route to it? Is the pull server on?
- Check the pull server web/IIS logs for errors to see if the pull server is having issues serving files. If the server isnt serving actions, youre obviously not able to get an action.
- Check that you can pull the configuration from the pull server manually using ```http://<server>/PSDSCPullServer.svc/Action(ConfigurationId='<guid>')/ConfigurationContent```.
it should download a copy of the MOF. If you can get a copy of the MOF there may be an issue with your configuration itself.
