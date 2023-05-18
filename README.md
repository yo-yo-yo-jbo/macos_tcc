# Introduction to macOS - TCC

Continuing with interesting security measures on macOS - here comes TCC.  
`TCC (Transparency, Consent and Control)` is a macOS mechanism aimed at protecting sensitive information.  
This includes access to user's private files (e.g. files on the Desktop), access to the camera and the microphone, location services access and many more.  
Interestingly, TCC protects those even against root-level attacks.

## The user experience aspect
Remember the Windoews Vista days? One of the things that really impacted the user experience was the introduction of UAC.  
Microsoft has concluded around that time that many attacks are run by administrator-level access, which essentially meant they could do anything.  
UAC was supposed to take care of that problem by splitting the administrator token to two parts, and creating UAC "prompts" when there's a need to run a privileged operation.  
The user experience for that was not great - approving every little operation began to feel like one of those idle clicker games, and so UAC was changed to have 4 different levels of enforcement - this is what we experience today in Windows 10\11.  
This isn't a UAC talk (perhaps I'll do one in the future!). Apple decided to do something similar - prompting the user when some application is going to access private data. However, Apple had a slightly different approach, as they decided to *persistently keep the user's choice*.  
That's excellent from a user-experience perspective, since the user has to approve only *once*, and then that choice is used forever (or until the user decides to reset their choice manually).  

