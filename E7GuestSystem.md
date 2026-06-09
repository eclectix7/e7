
## v1, Draft 

(see ~/e7/E7GuestSystem/E7GuestSystem.md for extended version)

Ok, let's write an alternate / new plan. We want to make a white-label version of the visitor system with the ability to sell subscriptions for different churches or businesses to use. 

Main Entities:
- account - entity accounts created as the main users of the system
	- account user - can make limited account modifications & view, manage, & message visitors 
	- account admin - extends the account user to control & modify all account options
- account location - a specific location, web URI, or form created for users to collect visitor data (not necessarily tied to a physical location)
- guest/visitor - people that visit user locations. They will open a link/scan a QR code to open the location visitor registration form
- system admin - system admin that manages the system and user accounts
- Message thread - Chat thread between a visitor and an account

Requirements: 
- Vite/Netlify web UI for account signup & management, visitor signup forms, & admin screens
- Expo (ios/android) mobile app to provide the same mobile interface for account creation & management, visitor management & messaging
- Convex back end
- Visitor messages sent via SMS service, Logged in Users engage through web or mobile app to protect their person phone numbers
- QR code that leads to the visitor opt-in registration form
	- account admin can configure what the visitor data forms
	- Forms will have options like party size, names, & other details, reason for visit, prayer request, followup request
- Users can reply with common phrases to opt out of followups or possibly data removal. Note: GDPR & similar law requirements related to churches vs businesses & physical location data collection require additional research

app main use cases / flow:
1. User (admin) creates a new account with Email+password, Google or apple oauth login
2. User creates location / visitor form, chooses from common form options (name(s), party count, local:boolean, moving:boolean, friends/family in church, reason for visiting) & configures automatic reply template (max 140 chars/msg), shares URI / QR code where visitors will find them
3. (optional) admin creates account user with invitation email, created account users can be granted admin privileges
4. Visitor scans QR code, registers as a visitor, opts in to followup messages
5. System saves visitor information for uses visitor information to send SMS welcome message using location admin template
6. System logs visitor activity & sends push notification to registered account users
7. account user opens app or web UI
	1. User can view contacts, activity/chat log, or notifications to continue any of the visitor conversations
	2. User adds arbitrary notes about visitor/contact
	3. User sets reminder to check back with individual visitor and/or regular reminder to check app for visitors to follow up with
	4. User updates contact information and status

The system will allow will provide a SaaS solution for churches and businesses that want to provide an easy way for visitors to register/sign up for follow up communications - **or** just just record having visited.  The users (paying subscribers) will be able to manage 1+ forms to collect data for potentially different intents & follow up with visitors that opt in for it through the web or mobile app.  Visitors will receive an semi-immediate response based on their data & an admin provided templates response for the form. Users (account subscribers) can set 0+ reminder notifications to help them remember to engage with visitors via the app.

## Kaizen Enhanced Version & Suggestions

