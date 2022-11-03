# Jira

The current SatSense Jira board is set up [here](https://satsense.atlassian.net/secure/RapidBoard.jspa?rapidView=9).   This board is actually the board for a project called "SatSense Devs", but it pulls in the issues for the other projects. The following details on how this board was set up, linking it to multiple projects, and integrating with Gitlab.  We assume that the user has set up [an account](https://www.atlassian.com/software/jira/free). 

# Projects

## Create Project (all except the "SatSense Devs" board)

To create a normal project, i.e. not one whose board that needs to be able to pull in issues from other projects:

* In the upper nav bar, click "Projects" and then "Create project".
* Click "Scrum" and then "Use Template", this will add the ability to add a backlog and sprints.
* Select "Team Managed" - whilst this has fewer features than "Company Managed", it is must easier to get going.
* Enter name and key and click "Create".

## Amend workflow

* Go to project board (click "Projects" and then select your project, the default view goes to the project board)
* Click three dots in top right, then "Manage workflow".
* Manage workflow - for example to add an "In Review" column, click "In-progress status" at the top and type to create.
* Save and close.

# SatSense Devs Project 

In this section we'll create a project that can pull in issues from other projects. To create a board for multiple projects you first need to create a filter.

* In the upper nav bar, click "Projects" and then "Create project".
* Click "Scrum" and then "Use Template", this will add the ability to add a backlog and sprints.
* Select "Company Managed" (compare with the above), this type of board will allow issues to be imported into this project.
* Enter name and key and click "Create".

## Create Filter

Once the project board is created, we need to amend the filter to allow access to other project issues.

* From top nav bar, click "Filters" and then "View all Filters".
* Click "Filter for <project name you just created> board" in top right.
* Click "Switch to JQL".
* Enter a statement, for example `project = POR OR project = API or project = AD ORDER BY Rank ASC` will pull in all the issues from the `POR`, `API` and `AD` projects.
* Click "Save"

## Map statuses

You might find that issues you expected to appear do not appear on your project board. This is likely to be a problem with unmapped statuses. Go to the board column settings:

* When viewing board, click three dots in top right and click "Board settings".
* Click "Columns" under "Settings" menu

You can add columns if needed (for instance, if you created a "In Review" column for your project, you might want to do it here also).

Click and drag statuses from the "Unmapped Statuses" column to the required column. You should now be able to see the issues as expected.

# Gitlab Integration

## Add gitlab to jira

* In nav bar, click "Apps" and then "Find new apps".
* Search for "gitlab.com for Jira Cloud" and follow the instructions.

## Create api token on jira

* Go to [https://id.atlassian.com/manage-profile/security/api-tokens](api token) page.  Rest is self explanatory. 

## Enable authorisation on gitlab

We currently have only done this on a per project basis (projects include satsense-portal, satsense-admin, satsense-domain, satsense-api, postgresingester, sat-tileserver)

* Go to project or group integrations - Whilst viewing project or group, click "Settings" then "Integrations".
* Click on "Jira" in the list.
* Fill in form, hints for settings:
    * "Web URL": `https://satsense.atlassian.net`
    * "Username or Email": `bob.ross@satsense.com`
    * "Password or API token": `<value created in previous section>`
