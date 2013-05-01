# GitHub Email API

The GitHub email API script reads an email from standard in and will do one of two things:

1. If the email is from `notifications@github.com` then it will change the `Reply-To` header into `me+base64-encoded-original-reply-to@example.org` and write the result to standard out.

	This means replies are sent to yourself instead of GitHub which allow us to process them before they go to GitHub.

2. If the email is to `me+base64-encoded-reply-to@example.org` then it will look for lines in the email matching these patterns:

		/^state:\s*(open|closed)/
		/^assignee:\s*(.*)/
		/^title:\s*(.+)/
		/^labels?:\s*(.*)/

	If any are found then it will update the issue (found by looking at the `Message-ID` header) with the corresponding value.

	In addition to updating the issue, it will resend the email to the original reply-to address after having removed all command lines, assuming the (non-quoted) body of the email is non-empty.

## Install

1. Clone this repository to your server:

		git clone https://github.com/sorbits/github-email-api.git

2. Install required gems (via [bundler](http://gembundler.com/)):

		bundle install

3. [Create a GitHub OAuth token](https://help.github.com/articles/creating-an-oauth-token-for-command-line-use):

		curl -u «user» -d '{"scopes":["repo"],"note":"GitHub Email API"}' https://api.github.com/authorizations

4. Create `~/.config/github-email-api`, e.g.:

		oauth_token: «token»
		email:       "me@example.org"
		log_file:    "~/logs/github-email-api.log"

5. Setup your mail server to run email through [`procmail`](http://www.procmail.org/).
6. Add the following to `~/.procmailrc`:

		# Handle GitHub notification email
		:0fw
		* ^From:.*\<notifications@github\.com\>
		| "/path/to/github-email-api"
	
		# Handle replies to a notification (me+…@example.org)
		:0w
		* ^To:.*\<me\+(.+)@example\.org\>
		| "/path/to/github-email-api"

		# Send an email if the above command failed
		:0e
		| (formail -rtI'From: "GitHub Email API" <me@example.org>' \
		    ; echo "Error processing reply, see log for details.") \
		| /usr/sbin/sendmail -oi -t -fme@example.org

## Updating Labels

Label names can be prefixed with a minus (`-`) to remove them. For consistency an optional plus (`+`) prefix is allowed when setting labels. Example:

	label: bug
	label: +bug
	label: -bug

Multiple labels can be specified by separating them with a comma or using multiple `label`/`labels` lines.

	labels: -too-vague, +crash

If you use a label name that doesn’t already exist, it will be created.

## Problems

* The `github-email-api` script will not retry incase of errors, e.g. if `api.github.com` return a `500 Internal Server Error`.

* Issues closed via email generally show “you closed this issue…” before the reply (presumably stating the reason) since the reply is added via (GitHub’s own) email API which has a noticeable delay.

* Setting up the GitHub email API is probably too complex for most users.
