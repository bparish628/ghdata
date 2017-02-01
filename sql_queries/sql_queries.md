# SQL Queries

The following queries are based on GHTorrent database.  The SQL is for all repos, but the addition
of a where clause will make any of them specific to a certain repo:



## Number of Members per Project


	SELECT count(DISTINCT project_members.user_id) AS num_members, projects.name AS project_name, url
	FROM project_members JOIN projects ON project_members.repo_id = projects.id
	GROUP BY repo_id




## Number of Contributors per Project:

GitHub defines contributors as those who have made "Contributions to master, excluding merge commits"

I do not see in the GHTorrent database schema a way to determine commits to master vs other branches,
Nor a way to differentiate merge commits from other commits.

Because of this, for the following SQL query I will define contributors as "users who have made a commit"

When viewing the table, one may notice that there is a separate author_id and committer_id for each commit
the commit author has written the code of the commit.
the commit committer has made the commit
example: author writes some code and does a pull request.  committer approves/merges the pull request

For the following SQL, I am considering the author to be the contributer.

	SELECT projects.name AS project_name, projects.url AS url, count(DISTINCT commits.author_id) AS num_contributers
	FROM commits 
	JOIN project_commits ON commits.id = project_commits.commit_id
	JOIN projects ON projects.id = project_commits.project_id
	GROUP BY project_commits.project_id
	
## total number of commits per project:

	SELECT count(commits.id) AS num_commits, projects.name AS project_name, projects.url AS url
	FROM commits 
		JOIN project_commits ON commits.id = project_commits.project_id
		JOIN projects ON projects.id = project_commits.project_id
	GROUP BY projects.id          
	
## Project Watchers

	SELECT count(user_id) AS num_watchers, projects.name AS project_name, url
	FROM watchers
		JOIN projects ON watchers.repo_id = projects.id
	GROUP BY projects.id


### Activity Level of Contributors

For the purposes of this SQL, I have defined it as the median number of commits to this repo per contributer.
I decided against mean because this could lead to skewing if one committer commits 1000 and the rest commit 1

Unfortunately, there is no median calculaton in SQL.

I have found a median calculation for SQL on stackoverflow and tested it on a test table.
Here it is, column and table names modified for clarity

http://dba.stackexchange.com/questions/2519/how-do-i-find-the-median-value-of-a-column-in-mysql
answered by Jeff Humphreys

	SELECT AVG(column_name) median 
	FROM(
	  SELECT x.column_name, SUM(SIGN(1.0-SIGN(y.column_name-x.column_name))) diff, count(*) nofitems, floor(count(*)+1/2)
	  FROM table_name x, table_name y
	  GROUP BY x.column_name
	  HAVING SUM(SIGN(1.0-SIGN(y.column_name-x.column_name))) = floor((COUNT(*)+1)/2)
	      OR SUM(SIGN(1.0-SIGN(y.column_name-x.column_name))) = ceiling((COUNT(*)+1)/2)
	) x;

The following query finds the total number of commits per repo per contributor.
This query will then be combined with the median calculation from above 
to determine median commits per repo per contributor (or it could be used
in combination with other methods of determining activity level)

	SELECT project_commits.project_id AS project_id, commits.author_id AS author_id, count(project_commits.commit_id) AS num_commits
		FROM commits
	    JOIN project_commits ON commits.id = project_commits.commit_id
	    JOIN projects ON projects.id = project_commits.project_id
	GROUP BY project_id, author_id

I am working on a combination query for the median of contributer activity, but so far my attempts take too long to run and MySQL server times out.

# Pull Requests

### Pull Requests Opened

	SELECT count(DISTINCT pull_request_id) AS num_opened, projects.name AS project_name, projects.url AS url
	FROM msr14.pull_request_history
	    JOIN pull_requests ON pull_request_history.pull_request_id = pull_requests.id
	    JOIN projects ON pull_requests.base_repo_id = projects.id
	WHERE action = 'opened'
	GROUP BY projects.id
	
### Pull Requests Closed

	SELECT count(DISTINCT pull_request_id) AS num_closed, projects.name AS project_name, projects.url AS url
	FROM msr14.pull_request_history
	    JOIN pull_requests ON pull_request_history.pull_request_id = pull_requests.id
	    JOIN projects ON pull_requests.base_repo_id = projects.id
	WHERE action = 'closed'
	GROUP BY projects.id

### Pull Requests Accepted

Assume that a pull request with a history record of being 'merged' has been accepted

pull_request table includes both head_repo_id and base_repo_id
base repo is where the changes will go
head repo is where the changes are coming from
http://stackoverflow.com/questions/14034504/change-base-repo-for-github-pull-requests

Since we are talking about the approval of pull requests, I will choose the base repo since that is where the changes are going.
	
Note: some of these results look unusual, in that projects that I would believe would be very active have few approved pull requests.

Possibly these groups do not use pull requests as often and edit master directly?

	SELECT count(DISTINCT pull_request_id) AS num_approved, projects.name AS project_name, projects.url AS url
	FROM msr14.pull_request_history
		JOIN pull_requests ON pull_request_history.pull_request_id = pull_requests.id
		JOIN projects ON pull_requests.base_repo_id = projects.id
	WHERE action = 'merged'
	GROUP BY projects.id
	


### Pull Requests Rejected

Assume that a pull request with a history record of being 'closed' but lacking one of being 'merged' has been rejected.

	SELECT count(DISTINCT pull_request_id) AS num_rejected, projects.name AS project_name, projects.url AS url
	FROM msr14.pull_request_history
		JOIN pull_requests ON pull_request_history.pull_request_id = pull_requests.id
		JOIN projects ON pull_requests.base_repo_id = projects.id
	WHERE action = 'closed' AND pull_request_id not in 
		(SELECT pull_request_id
		FROM msr14.pull_request_history
		WHERE action = 'merged')
	GROUP BY projects.id

## Total number of organizations by project making pull requests (approved or not):

	SELECT count(DISTINCT org_id) AS num_organizations, projects.name AS project_name, url
	FROM
		organization_members
	    JOIN users ON organization_members.user_id = users.id
	    JOIN pull_request_history ON pull_request_history.actor_id = users.id
	    JOIN pull_requests ON pull_request_history.pull_request_id = pull_requests.id
	    JOIN projects ON pull_requests.base_repo_id = projects.id
	WHERE pull_request_history.action = 'opened'
	GROUP BY projects.id

Alternately, using the "company" field in the users table instead of the organization:

	SELECT count(DISTINCT company) AS num_companies, projects.name AS project_name, url
	FROM
	    users
	    JOIN pull_request_history ON pull_request_history.actor_id = users.id
	    JOIN pull_requests ON pull_request_history.pull_request_id = pull_requests.id
	    JOIN projects ON pull_requests.base_repo_id = projects.id
	WHERE pull_request_history.action = 'opened' 
	GROUP BY projects.id
	
## Number of organizations by project making pull requests that are approved:

	SELECT count(DISTINCT org_id) AS num_organizations, projects.name AS project_name, url
	FROM
		organization_members
	    JOIN users ON organization_members.user_id = users.id
	    JOIN pull_request_history ON pull_request_history.actor_id = users.id
	    JOIN pull_requests ON pull_request_history.pull_request_id = pull_requests.id
	    JOIN projects ON pull_requests.base_repo_id = projects.id
	WHERE pull_request_history.action = 'opened'
		AND pull_requests.id in
	    (SELECT pull_request_id 
			FROM pull_request_history
		WHERE action = 'merged')
	GROUP BY projects.id


# Issues


### Number of Open Issues (current)

	SELECT count(DISTINCT issue_events.issue_id) AS num_open_issues, projects.name AS project_name, url AS url
	FROM msr14.issue_events
		JOIN issues ON issues.id = issue_events.issue_id
		JOIN projects ON issues.repo_id = projects.id
	WHERE issue_events.issue_id not in
		(SELECT issue_id FROM msr14.issue_events
		WHERE action = 'closed')
	GROUP BY projects.id



### Average Comments per Issue per Project

	SELECT avg(avg_num_comments), project_name
	FROM
	(
		SELECT count(comment_id) AS avg_num_comments, projects.name AS project_name, projects.id AS project_id
		FROM msr14.issue_comments
			JOIN issues ON issue_comments.issue_id = issues.id
			JOIN projects ON issues.repo_id = projects.id
		GROUP BY projects.id, issues.id
	) AS comments_per_issue
	GROUP BY project_id



### Amount of time a closed issue was open before closing (excludes open issues)

[Information about MySQL date handling](https://dev.mysql.com/doc/refman/5.5/en/date-and-time-functions.html)
	
Days open by issue:

	SELECT issue_id, open_date, closed_date, DATEDIFF(closed_date, open_date) AS days_open, projects.name AS project_name, url
	FROM
	(SELECT issues.id AS issue_id, issues.created_at AS open_date, issue_events.created_at AS closed_date, repo_id
	FROM msr14.issue_events
		JOIN issues ON issues.id = issue_events.issue_id
	WHERE action = 'closed') AS closed_issues
	JOIN projects ON projects.id = closed_issues.repo_id
	
Average days issue was open before closing by project:

	SELECT avg(days_open) AS average_days_til_issue_closed, project_name, url
	FROM
	(
		SELECT DATEDIFF(closed_date, open_date) AS days_open, projects.name AS project_name, url, projects.id AS project_id
		FROM
			(SELECT issues.id AS issue_id, issues.created_at AS open_date, issue_events.created_at AS closed_date, repo_id
			FROM msr14.issue_events
				JOIN issues ON issues.id = issue_events.issue_id
			WHERE action = 'closed') AS closed_issues
		JOIN projects ON projects.id = closed_issues.repo_id) AS issues_days_open
	GROUP BY project_id

## Average amount of time currently open issues have been open (excludes closed issues):
	
	SELECT avg(date_difference) / 365.25 AS average_in_years, avg(date_difference) AS average_in_days, project_name, url
	FROM
	(
		SELECT CURDATE() AS curr_date, open_date, DATEDIFF(CURDATE(), open_date) AS date_difference, issue_id, project_name, url, project_id
		FROM
			(SELECT DISTINCT issue_events.issue_id AS issue_id, projects.name AS project_name, url AS url, issues.created_at AS open_date, projects.id AS project_id
				FROM msr14.issue_events
					JOIN issues ON issues.id = issue_events.issue_id
					JOIN projects ON issues.repo_id = projects.id
				WHERE issue_events.issue_id not in
					(SELECT issue_id FROM msr14.issue_events
					WHERE action = 'closed')
			) AS open_issues
	) AS date_diffs
	GROUP BY project_id
	
## Issue tags by project (all tags):
A note on this: I see some unusual results, such as projects with many issues but no "bug" tags.
A possibility is that some communities handle tags differently from others, and may use the "bug" tags more heavily than others.
This may become a problem in future queries I will write specifically targeting the "bug" tag (in that the results may not
be a good comparison between repos)
	
	SELECT count(issue_id) AS num_issues_with_this_tag, repo_labels.name AS tag, projects.name AS project_name, url 
	FROM msr14.repo_labels
		JOIN projects ON repo_labels.repo_id = projects.id
		JOIN issue_labels ON issue_labels.label_id = repo_labels.id
	GROUP BY projects.id, repo_labels.id 

## Number of issues tagged as 'bug' by project:
	
	SELECT count(issue_id) num_bug_tags, repo_labels.name AS tag, projects.name AS project_name, url 
	FROM msr14.repo_labels
		JOIN projects ON repo_labels.repo_id = projects.id
		JOIN issue_labels ON issue_labels.label_id = repo_labels.id
	WHERE repo_labels.name = 'bug'
	GROUP BY projects.id, repo_labels.id 
	
## Average days an issue tagged with 'bug' exists until a project member comments:

	SELECT avg(time_to_member_comment_in_days) AS avg_days_to_member_comment, project_name, url
	FROM
	(
		SELECT DATEDIFF(earliest_member_comment, issue_created) time_to_member_comment_in_days, project_id, issue_id, project_name, url
		FROM
			(SELECT projects.id AS project_id, 
					MIN(issue_comments.created_at) AS earliest_member_comment, 
					issues.created_at AS issue_created, 
					issues.id AS issue_id, projects.name AS project_name, url
			FROM msr14.repo_labels
				JOIN projects ON repo_labels.repo_id = projects.id
				JOIN issue_labels ON issue_labels.label_id = repo_labels.id
				JOIN project_members ON projects.id = project_members.repo_id
				JOIN issues ON issue_labels.issue_id = issues.id
				JOIN issue_comments ON issue_comments.issue_id = issues.id
			WHERE repo_labels.name = 'bug'
				and issue_comments.user_id = project_members.user_id
			GROUP BY issues.id) AS earliest_member_comments) AS time_to_member_comment
	GROUP BY project_id


