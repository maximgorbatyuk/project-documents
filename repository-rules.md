# Repository rules

> If we are not agree with any rule here, we have to discuss the rule instead of ignoring it.

We are all humans, so we make mistakes and nobody is perfect. If any team member sees problems/issues with any rule written here, he should start a discussion why the rule is incorrect and should be changed.

> If the rule is incorrect and really should be changed, we will change the rule.

## GIT

1. Commit as often as it possible. Do not care about runner optimization or smth else. Commits represent your progress.
2. If you work with a ticket, you should name your branch with following patterns:
   - `feature/ticket-NNN` for features (`feature/PI-1`);
   - `bugfix/ticket-NNN` for bugfixes (`bugfix/PI-1`);
   - `hotfix/ticket-NNN` for hotfixes (bugfixes for production bugs) (`hotfix/PI-1`);
   - `task/ticket-NNN` for tasks that are not related to user stories but require some code changes (`task/PI-1`).
3. Merge code from parent branch as often as it possible. It will avoid merge conflicts in future.
4. Try to split big user stories in subtasks.
5. We should prefer `rebase` to `merge` operation.
6. We should avoid IDE settings in the repository. if you use another tool instead of Visual Studio, ypu have to add your IDE files in `.gitignore`.

## Code

1. All TODOs should start from words 'TODO <Name of the author>: <what's the issue and what should we do>'. The author's name will be useful to get more context if someone starts to implement his TODO.

## Code review

1. All teammates can do code-review for other teammates. Moreover, it will be great if all teammates take a participation in peer code-review session. Anyway, only maintainer can merge pull-requests.
2. If there are any misunderstands about code-review issues, an author of the code should explain his position and opinion. He all have bad mood sometimes, so code-reviewer sometimes can do or write wrong things. Anyway, teamleader is a responsible person for the project code base, so he has unlimited right to manage the code.

## Jira (or any other task board)

1. When a developer finishes his/her task, he moves the ticket into a 'Code-review' column.
2. When a code-reviewer finishes a code-reviewer session and if there are some issues that should be fixed, the code-reviewer moves the ticket into a 'In progress' column. The author of the code does not have to drop all other tasks in progress o start work on the ticket immediatelly. We all respect our time.

## Communication

1. We use Slack as a main communication channel but sometimes we get invitations by email, so we have to check our email inboxes.

## Meetings

1. If we are invited to a meeting, we have to respond on the invitation.
   - If we attend to the meetings, so we have to come to the meeting start time.
   - If the start time of the meeting is not compatible to our schedule, we have to respond "No" and purpose other time for the meeting.
2. If we miss a *daily statu*s meeting, we have to report the status information in a project thread:
   - What I did yesterday (on the last business day),
   - What I'm going to do today,
   - Do I see any problems / issues that can break our deadlines.
3. If we are not able to attend any project /non-project meeting, we have to notify our team about it as soon as possible.
4. If we are not able to work due to any exceptional occasions, we have to notify our Functional Manager (FM) and Project team (including Project Manager) during a day before.
5. If we are not able to work due to illness, we have to notify our Functional Manager (FM) and Project team (including Project Manager) during a day before or in a morning as soon as possible.
