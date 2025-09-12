# githubActions-org
# hi

curl -L \
  -X PUT \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer <YOUR-TOKEN>" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/OWNER/REPO/branches/BRANCH/protection \
  -d '{
    "required_status_checks": {
      "strict": true,
      "contexts": [
        "continuous-integration/travis-ci"
      ]
    },
    "enforce_admins": true,
    "required_pull_request_reviews": null,
    "restrictions": null
  }'