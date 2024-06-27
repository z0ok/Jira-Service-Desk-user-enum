# What is this all about?

Recently during a pentest project I've found an interesting way to enumerate Jira users via mentioning (tagging) in the Jira Software Service Desk application. The method is quite simple and obvious and was successfully executed in a killchain, so thx Atlassian!

I've reported this method and provided all requested information to Atlassian and got response:

> ... it has been concluded that this is considered expected behavior.
> 
> https://confluence.atlassian.com/jirakb/mention-group-of-users-using-feature-in-a-jira-issue-1346243947.html
>
> https://support.atlassian.com/jira-cloud-administration/docs/manage-global-permissions/
>
> Users with the Browse project permission can be searchable in the user picker field available on an issue.
>
> Granting Browse Project permission to users, makes them mentionable. In your example, the test user is able to search for other users, because those user have been given the Browse Project permission.

_Not applicable_. Welp, not thx Atlassian. Thus, I'm making this not-a-vulnerability note public.

# TLDR

1) Login to Service Desk application with Burp (or your preferred tool);
2) Create any issue from existing templates (just start process, no need to actually create it in the system);
3) In the Issue description mention any user;
4) Either intercept request `/rest/servicedesk/1/customer/participants/1337/user/search?q=test` or find it in the history and send to the Intruder;
5) Modify the `1337` param by hand or bruteforce it with Intruder;
6) Find non-empty responses;
7) Use projects with non-empty response to brute `q=test` param;
8) ???? 
9) PROFIT! You got a number of internal users. 

# And what is this Service Desk all about?

Service Desk is a Jira Software application that is installed on Jira to provide access to some functionality to clients without allowing them to view Jira itself. Link [here](https://www.atlassian.com/software/jira/service-management/features/service-desk). 

User/client/not-an-attacker who has access to Service Desk may create issue tickets in a way it has been set up by the instance administrators. Also, they can view their tickets and comments, and do all usual issue-stuff. There are some configuration restrictions, that prevent user/client/not-an-attacker from accessing/browsing all Jira Issues, Users, Apps, configurations, etc.

Btw, don't confuse Jira Service Desk with Jira Service Management.

# So, what's the trick?

Service Desk that I've tested (on-premises installation) had all restrictions mentioned above: Service Desk user was only allowed to browse his own issues and... that's all. Even mentioning other users was restricted. Freshly registered user could only create new issue tickets from template. 

And that's where the fun begins. Let's create a new user `not-hacker` and mess around a bit.

By default, all new users are assigned to some unknown (yet) project. When user tries to view issues - he gets nothing. Try to mention somebody - nothing. And for every operation (get or create issues, mention user, etc.) application makes request that contains project ID. 
If the user tries to mention `test` user, Service Desk makes a GET request to URL:

```
/rest/servicedesk/1/customer/participants/1337/user/search?q=test
```

`1337` part here is a project ID configured for the user to be default. This request returns an empty list `[]` and that's why `not-hacker` can't mention(tag) anybody from the application GUI.

Modify the request, bruteforce this ID and eventually you (and `not-hacker` of course) may find some misconfigured projects. As I later found out (after getting Jira instance admin privileges), `not-hacker` has no auto-groups or membership assigned. Furthermore, "compromised" project has `internal` security level.

As a result, when `not-hacker` makes the modified request, API returns first 10 users available for mentioning and suitable for our filter `q=test`. Besides it returns some extra info: `id`, `userKey`, `displayName`, `emailAddress`, `avatar` (link to gravatar) and `type` (user/admin). A super useful information for further attacks like password-spraying, bruteforce and social engineering.

Just bruteforce `q=[all_alphabet_letters]` and you will get a decent amount of internal Jira users data. That's it. 

At the end let's recall Atlassian's answer:

> Users with the Browse project permission can be searchable in the user picker field available on an issue.

All the users that can browse a project are automatically available for mentioning. Kinda weird permission setting, huh. That's why `not-hacker` is able to get user data from misconfigured projects by using not-an-IDOR not-a-vulnerability. 

Or maybe not ¯\\_(ツ)_/¯. I haven't done research "How it works internally", just got tasty users and continued the attack.
