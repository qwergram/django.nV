## Vulnerability Report
### *Reviewer 1:* Norton Pengra
### *Reviewer 2:* Kent Ross
### *Date:* April 22 2016

## Reviewing nVisium Task Manager

## Vulnerability 1

First of all, debug is set to True. So, that's kind of a problem.
When we force 404 errors, we can see every single URL route, which let's us
hit whatever we want there.

### *Injection*

We searched around forms.py to find that someone used `%s`.
We then tried the following: `test_pic', (select password from auth_user where username = admin), 9);--`

And got a `ProgrammingError` which refers to database operations, which means we successfully interacted with the database.
This is what the code originally said:
   `curs = connection.cursor()`
            `curs.execute(`
                `"insert into taskManager_file ('name','path','project_id') values ('%s','%s',%s)" %`
                `(name, upload_path, project_id))`
This is what the safer changed it to:
    `    file = File(`
                `name=name,`
                `path=upload_path,`
                `project=proj`
            `)`

### *Broken Auth*

After searching around the source code again, we discovered that the developer
created a `ModelForm` where the `Model` was the `User`. They only defined an `exclude` attribute
but failed to exclude `is_superuser` and `is_staff`. Which means any form submitted with injected html
that looks something like: `<input type="checkbox" name="is_staff"><input type="checkbox" name="is_superuser">`
will create an admin user that anyone can use. That's bad.

We can easily change the code in forms.py from:

```
class UserForm(forms.ModelForm):
    """ User registration form """
    class Meta:
        model = User
        exclude = ['groups', 'user_permissions', 'last_login', 'date_joined', 'is_active']

```

Replace the `exclude = [..]` with `fields = ['username', 'password', 'email', 'first_name', 'last_name']`

### *XSS*

XSS refers to people writing unwanted html on a page and then the page actually rendering it.
The instance we found was there was a `|safe` in the html templates after the username.
We then typed `<script>alert("I love waffles")</script>` as the username and sure enough,
a javascript alert popped up anywhere the username was displayed. The solution was to
remove the `|safe`.

### *IDOR Insecure Direct Object Reference*

In the source code, the  route does not actually check if the user owns
the task or project, meaning anyone who can guess the project id can edit and delete.
We can add `if request.user in task.user_assigned.all()` to the check if the user is
associated with the task at all.


### *Security Misconfiguration*

`DEBUG = True` That's how we found most of our urls. And 500 errors aren't masked.
We should set `DEBUG = False`, however I don't want to setup nginx or apache to serve
my static files, so I'll leave it to true for now.
