---
title:  "Applying GitLab labels automatically"
date:   2016-08-04 13:10:00 -0400
categories: devops GitLab services
excerpt: "Use automation to apply GitLab labels to optimize workflow."
---
<br>
![Services](/images/pharmacy-sticker-box.jpg)
<br>
<br>

# Automatic application of GitLab labels

In my [previous post]({% post_url 2016-08-03-gitlab-labels %}) I described how to use GitLab labels to easily triage.
[@shochdoerfer](https://twitter.com/shochdoerfer) asked me:

> @boc_tothefuture how can @gitlab labels be applied automatically for issues or merge requests?

# A quick webhook server
We use [GitLab webhooks](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/web_hooks/web_hooks.md) to automatically apply labels to incoming MRs.
To process the webhooks, we wrote a simple [webrick](https://en.wikipedia.org/wiki/WEBrick) server whose process is supervised by [runit](http://smarden.org/runit/)
and the incredibly well written [runit cookbook](https://github.com/chef-cookbooks/runit).

# Adding labels automatically
Using the Webrick as a base it's fairly easy to get labels added to your MRs when they are opened.
When the request comes in to your webrick server, look at the GitLab "object_kind" to see if its an MR.

{% highlight ruby %}
def merge_request?(request_body)
  body['object_kind'] == 'merge_request'
end
{% endhighlight %}

If the code is a merge request, the next step is calculate the labels that should be applied to the MR.  In our case
that is a 'Needs Review' label if the MR is just being opened.  Then because we use [semver](http://semver.org) and [thor-scmversion](https://github.com/RiotGamesMinions/thor-scmversion) we just scan all the commit messages for '#patch', '#minor' and '#major' to apply
the appropriate semver tag to the MR.

You will need a valid GitLab API key to modify and or request data about an MR.

{% highlight ruby %}
def update_labels(gitlab_server, api_key, request_body )
  project_id = request_body['object_attributes']['target_project_id']
  request_id = ['object_attributes']['id']
  labels = ['Needs Review'] if request_body['object_attributes']['action']
  semver_increment = semver_increment(gitlab_server, api_key,request_body )
  labels += semver_increment if semver_increment

  merge_data = {id: hook_id(hook), project_id: project_id(hook), labels: labels.to_a.sort.join(',')}
  url = "#{gitlab_server}/api/v3/projects/#{project_id}/merge_requests/#{request_id}?private_token=#{api_key}"
  RestClient::Request.execute(:method => :put, :payload => merge_data, :url => url)

end

def semver_increment(gitlab_server, api_key, request_body)
  from_branch = request_body['object_attributes']['target_branch']
  to_branch = request_body['object_attributes']['source_branch']
  project_id = request_body['object_attributes']['target_project_id']

  params = { private_token: api_key, from: from_branch, to: to_branch }
  url = "#{gitlab_server}/api/v3/projects/#{project_id}/repository/compare"
  changelog = JSON.parse(RestClient::Request.execute(:method => :get, :url => url, :headers => { params: params }))
  changelog = (changelog['commits'] || []).map { |commit| commit['message'] }
  return 'Major' if changelog.any? { |msg| msg.include? '#major' }
  return 'Minor' if changelog.any? { |msg| msg.include? '#minor' }
  return 'Patch' if changelog.any? { |msg| msg.include? '#patch' }
end

{% endhighlight %}


# Pro tips
You will need to do some extra work to make this work on updates.  For updates you will need pull the current labels and merge them where appropriate.  This is necessary
to update the labels when a subsequent commit to the MR takes it from a #patch to a #minor or #major.

You should thread your webrick server so the processing of the updating of incoming requests is in an different thread than the accepting of webhooks from GitLab.
If you don't do multi-thread you may run into issues where GitLab resends the webhooks because of a timeout.  The default timeout in GitLab is 10 seconds.  Simple threading with [ruby thread](http://ruby-doc.org/core-2.2.0/Thread.html) and a [queue](http://ruby-doc.org/core-2.2.0/Queue.html) should be sufficient.

Thanks to my colleague [Cameron McAvoy](https://www.linkedin.com/in/cameron-mcavoy-7515a35b) who added the label processing to the original webhook server I wrote.
