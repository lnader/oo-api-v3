---------------------------------------

**Use case**: Resource ownership  
As an organization I would like to know who created each resource. Also, I would like only the creator/owner of the resource to be able to view or edit the resource.

**Proposal**:
- Add an owner  field to each resource in Kubernetes.

**Example**: Project resource named "foo"

        POST /projects
        {
          "name": "foo"
        }

        HTTP/1.1 200 OK
        {
          "id": "54f4249f076573c0f4000222"
          "name": "foo",
          "owner": "jdoe"
        }

---------------------------------------

**Use case**: Give other users access to my resources  
As a user I want to be able to give access to my resources to another user.  I would like to limit the scope of this access.  For example, I may want to give only view access to one individual and editing rights to another.

**Proposal**:
- Add an accessible_to field to each resource in Kubernetes so users can give access to other users.

**Example**: Creating a project named "foo" and giving access to user "jdoe"

        POST /projects
        {
          "name": "foo"
          "accessible_to": [
                    { 
                       "id": "jdoe",
                       "type": "user",
                       "role": "edit",
                    }
        {

**Example**: Giving access to an existing project

        POST /projects/<project_id>/accessible_to
        { 
          "id": "jdoe",
          "type": "user",
          "role": "edit",
        }

---------------------------------------

**Use case**: Give access to groups of users  
As a user I want to give access to my resources to a group of users. I would like to limit the scope of this access. 

**Proposal**:  
- Create a new resource named "group".  This resource enables grouping of end-users into one entity for ease of assigning access control.

        POST /groups
        {
          "name": "engineering",
          "users": ["jdoe", "fbar"],
          "role": "user",
          "public": false
        }
     
- Use the accessible_to field described in the previous section to give access to this group.

        POST /projects
        {
          "name": "foo"
          "accessible_to": [
                      { 
                         "id": "53c4249f076573c0f4000001",
                         "type": "group",
                         "role": "view",
                      }
        }

---------------------------------------

**Use case**: Public groups  
As an enterprise customer I would like to create groups that are visible to all users in the organization.

**Proposal**:  
- Give admin access to a few users in the organization to create public resources

        POST /users
        {
          "id": "jdoe",
          "role" : "admin"
        }

-  Allow users with "admin" role to create public resources

        POST /groups
        {
          "name": "engineering",
          "users": ["jdoe", "fbar"],
          "role": "user",
          "public": true
        }

---------------------------------------

**Use case**: Public image repository  
As a member of the release management team in my organization I would like to create an image repository that is visible to other teams like QE, Development, etc.

**Example**: Accessible to all users

        POST /image_repository
        {
          "name": "Public Release Builds",
          "public": true
        }

**Example**: Accessible to only QE and Development groups

        POST /image_repository
        {
          "name": "Nightly Builds",
          "public": false
          "accessible_to": [
                      { 
                         "id": "53c4249f076573c0f4000001",
                         "type": "group",
                         "role": "view",
                      },
                      { 
                         "id": "54f4249f076573c0f4002222",
                         "type": "group",
                         "role": "edit",
                      }
        }

---------------------------------------

**Use case**: Give access to 3rd party services  
As a user I would like to give access to an external build service to post new images to my image repository. I would like this access to be temporary and limited in scope. 

**Proposal**:  
- Add a new resource for obtaining an authorization code per [OAuth-2.0-section-4.1](http://tools.ietf.org/html/rfc6749#section-4.1)
- Add a new resource for obtaining access tokens per [OAuth-2.0-section-4.1.3](http://tools.ietf.org/html/rfc6749#section-4.1.3)

---------------------------------------

**Use case**: Re-assign resources for terminated user  
As a company, when a user leaves I would like to re-assign his resources to another user or group, so his resources can continue to operate after his departure.

**Proposal**:  
- Give admin access to a few users in the organization to re-assign resources as needed

        POST /users
        {
          "id": "jdoe",
          "role" : "admin"
        }

-  Allow users with "admin" role to  re-assign resources as needed

**Example**: Add new user to project

        POST projects/<project_id>/accessible_to
        {
          "id": "jdoe",
          "type": "user",
          "role": "owner",
        }

**Example**: Remove user from project

        DELETE projects/<project_id>/accessible_to/<user_id>

---------------------------------------

**Use case**: Terminating user  
As an administrator in an enterprise I would like to be able to delete users when they leave the company.

**Proposal**:  
- Give admin access to a few users in the organization to delete users

        POST /users
        {
          "id": "jdoe",
          "role" : "admin"
        }

-  Allow users with "admin" role to delete users

        DELETE /users/jdoe

---------------------------------------

**Use case**: Corporate accounts  
As  a company I would like to create a corporate account and be responsible for the billing.  However, I would like to give access to individual  users or groups in my company to resources available to this account.

**Proposal**:  
-  Add a new resource called "account"

        POST /accounts
        {
          "name": "Red Hat",
          "code" : "RHT123",
          "primary_contact": "jdoe@redhat.com"
        }

        HTTP/1.1 200 OK
        {
          "id": "54f4249f076573c0f4000222"
          "name": "Red Hat",
          "code" : "RHT123"
        }

-  Allow users to specify an account code when registering or being added by an admin.

        POST /users
        {
          "id": "jdoe",
          "role" : "user",
          "account_code": "RHT123";
        }

        HTTP/1.1 200 OK
        {
          "id": "jdoe",
          "account_id": "54f4249f076573c0f4000222",
        }

---------------------------------------

