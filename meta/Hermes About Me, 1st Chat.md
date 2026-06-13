- Hello Hermes, my name is Todd
- I would like to call you Kaizen
- I am a senior full stack architect & developer with experience with LAMP, Node, React, jQuery, Expo
- I have built 100+ applications like Facebook, Instagram, or Youtube in the last 20 years
- I prefer OO languages & enjoy Design Patterns, SOLID & GRASP coding principles
- I have a startup named E7 LLC
- We are in phase one of the startup (no revenue)
- We have deployed 4 productivity Android apps 
- We will be able to publish on iOS soon, a new macbook arrives in a few days
- I use Cursor as my main coding IDE (not on this device)
- I also have a Codex subscription (not connected here)
- I have tried using the Pi agent/editor and want to get it setup better
- I have not advertised the apps yet because they are not published on Apple also
- I have many other apps planned to use AI to help people with different problems
- The plan is to create multiple revenue streams via apps that solve people's problems and help them and then use that money to find more ways to help more people, provide job & learning opportunities for people, and offer assistance to people in need
- We are still working on long term strategy and plan
- This device has been dedicated to you (Hermes) as an assistant
- I study & enjoy mathematics, physics, quantum mechanics, relativity theory, chaos theory, dynamic systems, coffee, tea, neuro-hacking, bio-hacking, architecture, techno music, many cultures especially Japanese Samurai/Bushido, I enjoy languages 

## Follow Up

Great, based on everything you know about be I want you to tell me how you can help me accomplish our goals.
- Not that this computer is not intended for development & not connected to the projects for security reasons.
- The repos and Cursor are set up on other systems used exclusively for development. I'm not sure how much access to critical accounts I want to add to a system under full agent control.
- Tell me how you think we should work together. What use cases can we tackle or plan together to maximize efficiency & value added. 
- We have a LAMP website used for a central hub:
	- public facing app pages required by app stores like: terms of service, privacy policy, account/data deletion request etc.
	- Central app feedback APIs & dashboards & other app features
- We use Convex for some application back end systems and API proxies (to keep keys out of the code bases)
- We use Expo for mobile app development
	- Our apps currently range from Expo SDK 52 - 55
	- Expo SDK 56 was just release last week
- We work with Cursor for development
	- The apps are either use ads or in-app purchases or both
	- They are generally traditional apps augmented with AI features
	- We have had difficulty with converting our Clerk app from the Dev instance to the Production instance due to trouble with the Google credential connection breaking everything when trying to set it up, so we're not a fan of Clerk / Authentication & prefer to avoid needing it until that is figured out
- We want to improve / streamline some of the tedious parts of the application development.
	- App store optimization
	- Marketing content / Copy writing
	- Marketing / app image & video creation
	- Organizing and planning centralized services that can be share by apps with common requirements (private APIs)
- I need you to give me specific activities and benefits that you can help with. I don't want a generic list of things LLMs can do - i already know them. I want a list of lucid well-informed ideas & plans based on what information has been shared. Please ask clarifying questions before responding if needed.

## Followup Questions

1.  Security Boundaries: To what extent are you comfortable sharing anonymized or scaffold-level code (e.g., class structures, interface definitions) here? I can provide much higher value if I can see the structure of your shared API proxy code, provided you strip any secrets/keys beforehand.
2.  Infrastructure Orchestration: Do you want me to monitor your backend status? For example, I could periodically query health check endpoints on your LAMP hub or Convex dashboard and report issues to you here.
3.  Task Management: How are you currently tracking your development lifecycle? Do you have an existing Kanban board or ticket management system (Linear, Jira, GitHub Issues) that I should hook into, or should we set up a lightweight one internally?
4.  Expo SDK Upgrade Strategy: SDK upgrades can be brutal. Do you want me to draft a migration checklist based on the Expo 56 release notes specifically for your app architecture to ensure your transition from 52-55 is smoother than a manual upgrade?

1 - I'm not sure yet. I'm been thinking of setting up a communications channel for indirect agent-agent messaging with HITL approval in such situations
2 - Yes, at some point I want to connect you to various data streams so that you can alert me in real time here or via Telegram if needed, compile data or formulate responses, etc automatically. We want to keep health check endpoints on our systems so we can monitor them.
3 - No, each project uses manual markdown todo lists. One of the earlier things we want to do it build a central Jira-like repo to track app planning or features, bugs, & backlog with our own Kanban board to monitor them. We are also going to build an internal app used to view and manage all the app states, todo lists, etc in addition to capturing & updating features and plans via mobile.
4 - No, we don't want to focus on that right now.

1.  Management Dashboard: For the internal management app you want to build, do you have a preference for the tech stack? Given you are experienced in React/Expo, should we aim for a web-based portal using those same patterns?
2.  Notification Integration: Please run send_message(action='list') so I can see which platforms (Discord, Telegram, etc.) are available for me to use for these autonomous alerts.
3.  Communication Channel: For the "agent-to-agent" setup, what is your preferred mechanism? Are we talking about a shared database/repo on Gitea/GitHub, or something more direct?
    
How would you like to start?
*   Option A: I draft the OpenAPI specification for your E7 Central Management API.
*   Option B: I inventory your current notification capabilities and set up a prototype health check cronjob for your LAMP hub.
*   Option C: We formalize your project planning workflow with the markdown templates I mentioned.
    
Which holds the highest priority for you right now, Kaizen?

I had to interrupt because I expected shift+enter to allow me to keep typing, but it sent you can incomplete answer. Here is the full response:

1 - We want both LAMP (browser) dashboard access and a mobile app, so they will provide two views to work with the underlying data.
2 - I don't know what you mean "run send_message(action='list')" we set up Telegram
3 - I was planning on adding our own Db & APIs on the LAMP server or in Convex

We actually don't want to start these just yet. This was mostly to give you long term context &  see what your ideas would be & decide what the plan is.  We will need the skills

---

This sounds good, but you need to keep the content about me & high level state info about E7 also & probably a reference to the details in case they're relevant to a future conversation because future conversations will have dependencies or relationships with what has been discussed.

Yes, I would like you to draft the roadmap.
Also note that I created an Obsidian folder for you at ~/e7/kaizen/


- [x] Setup Hermes Kanban with main items from above
- [ ] Setup suggested agent skill chron job

