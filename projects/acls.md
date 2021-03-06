---
title: Repository permissions in Sourcegraph
description: support repository permission restrictions
authors:
  - Beyang Liu
startdate: 2018-10-10
releasedate: yyyy-mm-dd
publishdate: yyyy-mm-dd
published: false
---

# Repository permissions in Sourcegraph

## Background/context

Prospective customers have requested repository permissions restrictions (colloquially "ACLs") in
Sourcegraph. The lack of permissions synced from either code host or SSO identity provider is
preventing multiple customers from adopting Sourcegraph.

An ancillary benefit of ACLs is that it would require users to sign in to a given Sourcegraph
instance, which means Sourcegraph instance admins could see more useful metrics about how that
Sourcegraph instance is being used by developers.

## Customer questionnaire

Please fill out this questionnaire: https://goo.gl/forms/ljcbTFGCkjM0sA2K2

## Permissions data model

Customers, admins, and users should evaluate this data model to determine if it is powerful enough
to support the repository permissions they would like to enforce. See the ["Scenarios"
subsection](#scenarios) for walkthroughs of this data model as it would apply to a selection of
concrete setups.

The proposed design breaks repository permissions into three interfaces:

```go
type (
    UserID  string
    AuthzID string
)

type Perm string

const (
    PermRead Perm = "read"
)

// AuthnProvider supplies the current user's canonical ID. The canonical ID is that
// which uniquely identifies the user in the identity provider, the source of truth
// for identity in the deployment environment. The identity provider can be any of
// the following:
//
// * SAML identity provider
// * OpenID Connect identity provider
// * Code host (e.g., GitHub.com, GitLab.com) if it is used to sign into other services
// * Other SSO identity providers like LDAP, etc.
//
// The identity provider should also be the sign-in mechanism for Sourcegraph.
//
// In most cases, the AuthnProvider will just return the current user's Sourcegraph
// username (which will be the same username provided by the identity provider).
type AuthnProvider interface {
    CurrentIdentity(ctx context.Context) UserID
}

// AuthzProvider is the source of truth for which repositories a user is authorized to view.
// The user is specified as an AuthzID, which identifies the user to the AuthzProvider.
// In most cases, the AuthzID is equivalent
// to the UserID supplied by the AuthnProvider, but in some cases, the authorization source of
// truth has its own internal definition of identity which must be mapped from the authentication
// source of truth. The IdentityToAuthzID interface handles this mapping. Examples of
// authorization providers include the following:
//
// * Code host
// * LDAP groups
// * SAML identity provider (via SAML groups)
//
// In most cases, the code host is the source of truth for repository permissions and therefore
// the code host is the authorization provider.
type AuthzProvider interface {
    // RepoPerms returns a map where the key set is the subset of the input repos to which the
    // user identified by authzID has access. The values of the map indicate which permissions
    // the user has for each repo.
    //
    // Discussion note: this is a better interface than ListAllRepos, because in some cases,
    // the list of all repositories may be very long (especially if the returned list includes
    // public repos). RepoPerms seems like a sufficient interface for all use cases inside
    // Sourcegraph and leaves up to the implementation which repo permissions it needs to compute.
    // In practice, most will probably use a combination of (1) "list all private repos the user
    // has access to" and (2) a mechanism to determine which repos are public/private.
    RepoPerms(ctx context.Context, authzID AuthzID, repos []api.RepoURI) (map[api.RepoURI]map[Perm]bool, error)
}

// IdentityToAuthzIDMapper maps canonical user IDs (provided by the AuthnProvider) to AuthzProvider
// IDs. In other words, it maps identity provider usernames to authorization provider usernames.
//
// In most cases, the identity provider username should be equivalent to the authorization provider
// username. However, it is not guaranteed. For instance, some code hosts may have a different
// internal username than the username supplied by the SSO login service.
//
// It is recommended to keep this interface as simple as possible. E.g., don't return an access token for the authzID,
// because it's good to give that responsibility to the AuthzProvider in case API rate limits
// are a concern.
type IdentityToAuthzIDMapper interface {
    // AuthzID returns the authzID to use for the given AuthzProvider. This will
    // be the identity function in most cases. Returns the empty string if no authz identity
    // matches.
    AuthzID(u UserID, a AuthzProvider) AuthzID
}

```

Together, implementations of these three interfaces will be used to determine whether the current
user has access to a given repository in each HTTP request. The `AuthnProvider` will provide the
user ID of the current user, the `IdentityToAuthzIDMapper` will map the user ID to a username
understood by the `AuthzProvider` and the `AuthzProvider` will return the list of repositories the
current user has access to.

Package `authz` will contain the three interface definitions above and also registration functions:

```go
package authz

func RegisterAuthnProvider(p AuthnProvider)
func RegisterAuthzProvider(p AuthzProvider)
func RegisterIdentityToAuthzIDMapper(m IdentityToAuthzIDMapper)
```

Package `conf` contains the configuration types for code hosts and identity providers. Based on the
site configuration fields set, the `conf` package will call the appropriate `authz` registration
functions. The following fields will be added to the site config schema:

```go
type GitHubConnection struct {
    ...
    AuthzProvider	bool
    ...
}


type GitLabConnection struct {
    ...
    AuthzProvider	bool
    ...
}
```

The `AuthzProvider` field, if true, instructs Sourcegraph to treat the code host as the
authorization source of truth. Note that the actual `AuthzProvider` used will depend on whether or
not the connection is to an on-prem or cloud code host instance. Some configurations may be
unsupported initially and Sourcegraph will throw an error on startup in those cases. The "Scenarios"
section below walks through specific use cases and how they will be supported under the proposed
model.


### Scenarios

#### GitLab on-prem + SAML

Configuration:

```json
{
    "auth.providers": [
        {
            "type": "saml",
            "displayName": "My SAML ID provider",
            "identityProviderMetadataURL": "https://sso.mycompany.com/saml"
        }
    ],
    "gitlab": [
        {
            "url": "https://gitlab.mycompany.com",
            "token": "<GitLab impersonation token>",
            "authzProvider": true,
        }
    ]
}
```

* `AuthnProvider.CurrentIdentity` returns the username of the current user, which is the same as the
  SAML username.
* `IdentityToAuthzIDMapper.AuthzID` is the identity function (it just returns the username as the
  AuthzID). This assumes that the GitLab username is the same as the SAML userame.
* `AuthzProvider.RepoPerms` uses the [GitLab impersonation
  token](https://docs.gitlab.com/ee/api/#impersonation-tokens) to [list all projects accessible to
  the username](https://docs.gitlab.com/ee/api/projects.html#list-all-projects).


#### GitHub business cloud + SAML

Configuration:

```json
{
    "auth.providers": [
        {
            "type": "saml",
            "displayName": "My SAML ID provider",
            "identityProviderMetadataURL": "https://sso.mycompany.com/saml"
        }
    ],
    "github": [
        {
            "url": "https://github.com",
            "token": "<personal access token with access to all repositories to be indexed by Sourcegraph>",
    		"authzProvider": true
        }
    ]
}
```

* ~`AuthnProvider.CurrentIdentity` returns the username of the current user, which is the same as the
  SAML username.~
* ~`IdentityToAuthzIDMapper.AuthzID` is the identity function.~
* ~`AuthzProvider.RepoPerms` calls the following endpoints to get the list of accessible
  repositories:~
  * ~[List user repositories](https://developer.github.com/v3/repos/#list-user-repositories)~
  * ~[List organization repositories](https://developer.github.com/v3/repos/#list-organization-repositories)~
  * ~[List user organizations](https://developer.github.com/v3/orgs/#list-user-organizations)~
  
  ~Note that this may omit some repositories the user has access to, including repositories
  accessible to an organization where the user's membership is not public and repositories that a
  user has been granted access to as an external collaborator. The GitHub API only permits listing
  all repositories accessible to a user via a request that is authenticated as the user in question.
  In practice, that means we must obtain an OAuth token for every user. In the case of GitHub
  business cloud with SAML sign-in, however, it's unclear whether we have a way of obtaining that
  OAuth token (or if we do, if it would require double sign-in). If we can obtain an OAuth token,
  then this case reduces to that of vanilla GitHub.com (see below). If not, we will need to resort
  to the implementation above.~

Upon further investigation, it seems that GitHub.com Business Cloud with SAML SSO supports
third-party OAuth apps in more or less the same way as vanilla GitHub.com with native auth.  So in
this setup, Sourcegraph should treat GitHub as the source of identity (even though GitHub in turn
treats the SSO provider as the source of identity).

It reduces to the vanilla GitHub.com case (see below): Sourcegraph uses GitHub OAuth as a sign-in
mechanism. It uses the user OAuth token to make API requests to determine what the user-repo
permissions are.


#### GitHub.com

In order to support GitHub.com repository permissions, we require that GitHub.com be the sign-in
mechanism for the Sourcegraph instance. This is necessary to obtain the OAuth token used in the
GitHub API requests for the list of repositories accessible to a user.

Configuration:

```json
{
    "auth.providers": [
        {
            "type": "oauth",
            "displayName": "GitHub.com",
            "issuer": "https://github.com"
        }
    ],
    "github": [
        {
            "url": "https://github.com",
            "authzProvider": true
        }
    ]
}
```

* `AuthnProvider.CurrentIdentity` returns the username of the current user, which is the same as the
  GitHub username.
* `IdentityToAuthzIDMapper.AuthzID` is the identity function.
* `AuthzProvider.RepoPerms` calls the [List your
  repositories](https://developer.github.com/v3/repos/#list-your-repositories) endpoint using the
  current user's OAuth token.


#### GitHub Enterprise + SAML

Configuration:

```json
{
    "auth.providers": [
        {
            "type": "saml",
            "displayName": "My SAML ID provider",
            "identityProviderMetadataURL": "https://sso.mycompany.com/saml"
        }
    ],
    "github": [
        {
            "url": "https://github.mycompany.com",
            "token": "<GitHub Enterprise impersonation token>",
            "authzProvider": true
        }
    ]
}
```

* `AuthnProvider.CurrentIdentity` returns the username of the current user, which is the same as the
  SAML username.
* `IdentityToAuthzIDMapper.AuthzID` is the identity function. This assumes that the GitHub username
  is the same as the SAML userame.
* `AuthzProvider.RepoPerms` uses the [GitHub Enterprise impersonation
  token](https://developer.github.com/enterprise/2.14/v3/enterprise-admin/users/#create-an-impersonation-oauth-token)
  to [list all repositories accessible to the
  user](https://developer.github.com/v3/repos/#list-your-repositories).

#### Other situations

This section summarizes implementation notes for other `AuthzProvider` scenarios.

- Bitbucket Server
  - Does not yet support impersonation tokens, inquiry posted here: https://community.developer.atlassian.com/t/any-plans-to-support-impersonation-tokens/23062
  - If we add support now, we'll have to use techniques similar to what's described in the GitHub
    business cloud scenario.
- Gitolite: Gitolite permissions can be fetched directly via the SSH API.
- AWS CodeDeploy
  - No impersonation token, but seems like it should be possible through IAM + CodeCommit API.
- GitHub.com, Bitbucket.org, GitLab.com
  - No impersonation token.
  - Each of these is an OAuth provider, so we would require login to Sourcegraph be done via OAuth
    through the code host.
- LDAP
  - Need more specific customer use case feedback to move forward here.

In situations where the implementation of `AuthzProvider.RepoPerms` is costly (e.g., due to
multiple API requests or API limits), it is acceptable to introduce some amount of caching to the
`AuthzProvider`, meaning that repository permissions on Sourcegraph may lag those in the
authorization provider by some amount of time (likely minutes to hours).

### Known concerns/risks

* It would be simpler for the config schema to omit the `authzProvider` field and just assume the
  admin would like to turn on ACLs for things like GitHub Enterprise and GitLab on-prem. Doing so,
  however, would elevate the access token from normal personal access token to impersonation token
  (effectively a "sudo"-level token). This may retard adoption. It may also break existing instances
  that currently user a normal personal access token. Would the greater config simplicity justify
  removing the `authzProvider` or should we keep it?

## Implementation notes

> Note: this section will likely only be of interest to Sourcegraph developers. Customers, end
> users, and Sourcegraph admins may skip unless they are interested in the implementation details.

Within the Sourcegraph backend, authz checks should satisfy the following conditions:
* Add minimal overhead to request latency.
  * Corollary: There is an acceptable amount of lag (tunable) between changes to ACLs on the code host and those changes being enforced on Sourcegraph.
* Authz should be enforced in any application request for repository info, including search.
* Authz behavior should be well-tested.

Checks should be invoked in the `db` layer (the `repos.getBySQL` and `repos.List` methods). The
assumption here is that there is no code path that allows a request to return repository data
without calling into `repos.getBySQL` or `repos.List` in the `db` package.

The authz checks, interfaces, and data structures will live in the `authz` package, which will be
included in the `repo-updater` service. Each frontend HTTP request cycle will involve one additional
HTTP request to `repo-updater` to check authz. `repo-updater` is a singleton, so we will be able to
cache authorization provider API responses effectively.

If redundant authz-related HTTP requests to `repo-updater` adds significant performance overhead, we
can introduce a cache into the authz client.
