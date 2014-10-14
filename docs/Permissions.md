#Xataface Permissions

Xataface includes a fine-grained, extensible, expressive permissions infrastructure that allows you (the developer) to define exactly who gets access to which actions in which context.

You can define permission rules for your application at 4 different levels:

1. **Application Level** By defining a `getPermissions()` method in the Application Delegate Class.
2. **Table Level** By defining a `getPermissions()` method in your table delegate classes.
3. **Record Level** By using case-by-case logic inside your `getPermissions()` method.
4. **Field Level** By defining `fieldname__permissions()` methods in your table delegate classes.

It is recommended that you define very restrictive permissions at the application level, then selectively remove restrictions at the table level as required to provide users access to only those areas that they need.

##Example Permission Configurations

In order for permissions to be meaningful, your application should probably have authentication enabled.  So assume that authentication is enabled, and your database has a `users` table defined as follows:

~~~
CREATE TABLE `users` (
   username VARCHAR(50) NOT NULL PRIMARY KEY,
   password VARCHAR(128),
   role ENUM('USER','ADMIN') DEFAULT 'USER',
   email VARCHAR(255)
)
~~~ 

And that your `conf.ini` file contains the following:

~~~
[_auth]
   users_table=users
   username_column=username
   password_column=password
   email_column=email
~~~


###Full Access to Administrator, No Access Otherwise

In your Application Delegate class (i.e. `conf/ApplicationDelegate.php`):

~~~
function getPermissions(Dataface_Record $record = null){
   $user = Dataface_AuthenticationTool::getInstance()->getLoggedInUser();
   if ( $user and $user->val('role') === 'ADMIN' ){
      return Dataface_PermissionsTool::ALL();
   } else {
      return Dataface_PermissionsTool::NO_ACCESS();
   }
}
~~~


###Regular Users have READ-ONLY access to the "products" table

In your Application Delegate class (i.e. `conf/ApplicationDelegate.php`) you disallow access to everyone except ADMIN users:

~~~
function getPermissions(Dataface_Record $record = null){
   $user = Dataface_AuthenticationTool::getInstance()->getLoggedInUser();
   if ( $user and $user->val('role') === 'ADMIN' ){
      return Dataface_PermissionsTool::ALL();
   } else {
      return Dataface_PermissionsTool::NO_ACCESS();
   }
}
~~~

Then in your *products* delegate class (i.e. `tables/products/products.php`) you provide read-only access to regular users.

~~~
function getPermissions(Dataface_Record $record = null){
   $user = Dataface_AuthenticationTool::getInstance()->getLoggedInUser();
   if ( $user and $user->val('role') === 'USER' ){
      return Dataface_PermissionsTool::READ_ONLY();
   } else {
      return null;
   }
}
~~~

**Notice** that we return null for all users except for logged in users with *role*="USER".  This means that, for other users, the default permissions (defined in the Application Delegate Class) should be used.

###Regular Users have EDIT access to their Own Product Records

In order to provide this type of functionality, we will assume that we have a mechanism to determine whether a particular user is the "owner" of a particular product record.  For this example, we will use the following mechanism:

1. The `products` table has an `owner` field with the username of the user that owns it.
2. We define a `beforeInsert` trigger in the `products` table to set the `owner` field to the currently logged-in user.

I.e. Assume the `products` table is defined as follows:

~~~
CREATE TABLE products (
    product_id INT(11) NOT NULL auto_increment PRIMARY KEY,
    product_name VARCHAR(128),
    owner VARCHAR(50)
~~~

Also we define the `beforeInsert` trigger in the `products` delegate class (i.e. `tables/products/products.php`:

~~~
function beforeInsert(Dataface_Record $record){
   $user = Dataface_AuthenticationTool::getInstance()->getLoggedInUser();
   if ( $user and !$record->val('owner') ){
       $record->setValue('owner', $user->val('username'));
   }
}
~~~

Now we can define our permissions in the `products` table delegate class (i.e. `tables/products/products.php`:

~~~
function getPermissions(Dataface_Record $record = null){
   $user = Dataface_AuthenticationTool::getInstance()->getLoggedInUser();
   if ( $user and $user->val('role') === 'USER' ){
      if ( $record and $user->val('username') === $record->val('owner') ){
          return Dataface_PermissionsTool::getRolePermissions('EDIT');
      } else {
          return Dataface_PermissionsTool::READ_ONLY();
      }
   } else {
       return null;
   }
}
~~~

There are two small wrinkles in this implementation, however:

1. Regular users only have READ-ONLY permission in the `products` table so they will never be able to create products that they own.
2. Product owners have full edit privileges to all fields of the `products` table so they could theoretically change the owner.  We may want to prevent this.

In order to resolve the first issue, we'll add the `new` permission to the regular user permissions.  For the 2nd issue, we'll add a `owner__permissions` method in the `products` table delegate class to disallow editing by owners. 

So the changed permission methods in the `products` delegate class is:

~~~
function getPermissions(Dataface_Record $record = null){
   $user = Dataface_AuthenticationTool::getInstance()->getLoggedInUser();
   if ( $user and $user->val('role') === 'USER' ){
      if ( $record and $user->val('username') === $record->val('owner') ){
          return Dataface_PermissionsTool::getRolePermissions('EDIT');
      } else {
          $perms = Dataface_PermissionsTool::READ_ONLY();
          $perms['new'] = 1;
          return $perms;
      }
   } else {
       return null;
   }
}

function owner__permissions(Dataface_Record $record = null){
   $user = Dataface_AuthenticationTool::getInstance()->getLoggedInUser();
   if ( $user and $user->val('role') === 'USER' 
           and $record and $user->val('username') === $record->val('owner')
           ){
      return array('edit' => 0);
   } else {
      return null;
   }
}
~~~


In these examples we were introduced to 3 new concepts:

1. The `Dataface_PermissionsTool::getRolePermissions()` method.
2. That the `Dataface_PermissionsTool` methods simply return an associative array assigning `0` or `1` to each permission.
3. The `fieldname__permissions()` method.

##Using `Dataface_PermissionsTool` To Get Permissions

In the above examples, we have used the `Dataface_PermissionsTool` class to return the permissions that should be granted to a user.  We have seen, the following methods so far:

* `DatafacePermissionsTool::ALL()` : Returns *ALL* permissions that have been defined for the application.
* `DatafacePermissionsTool::NO_ACCESS()` : Returns an array mask that includes all of the defined permissions set to zero.  I.e. it disables ALL permissions.
* `Dataface_PermissionsTool::READ_ONLY()` : Returns all permissions related to reading data, but does not include permissions related to editing, adding, or deleting data.
* `Dataface_PermissionsTool::getRolePermissions()` : Returns all of the permissions from a specified named group of permissions.  

All of these methods essentially just return associative arrays mapping permission names to boolean (more correctly *zero-one*) values. As an exercise, you may want to try a `print_r()` on the results of these methods just to see what they look like.  I.e.:

~~~
print_r(Dataface_PermissionsTool::ALL());
~~~

would result in something like:

~~~
Array
(
    [view] => 1
    [link] => 1
    [view in rss] => 1
    [list] => 1
    [calendar] => 1
    [edit] => 1
    [new] => 1
    [select_rows] => 1
    [post] => 1
    [copy] => 1
    [update_set] => 1
    [update_selected] => 1
    [add new related record] => 1
    [add existing related record] => 1
    [delete] => 1
    [delete selected] => 1
    [delete found] => 1
    [show all] => 1
    [remove related record] => 1
    [delete related record] => 1
    [view related records] => 1
    [view related records override] => 1
    [related records feed] => 1
    [update related records] => 1
    [find related records] => 1
    [edit related records] => 1
    [link related records] => 1
    [find] => 1
    [import] => 1
    [export_csv] => 1
    [export_xml] => 1
    [export_json] => 1
    [translate] => 1
    [history] => 1
    [edit_history] => 1
    [navigate] => 1
    [reorder_related_records] => 1
    [ajax_save] => 1
    [ajax_load] => 1
    [ajax_form] => 1
    [find_list] => 1
    [find_multi_table] => 1
    [register] => 1
    [rss] => 1
    [xml_view] => 1
    [view xml] => 1
    [manage_output_cache] => 1
    [clear views] => 1
    [manage_migrate] => 1
    [manage] => 1
    [manage_build_index] => 1
    [install] => 1
    [expandable] => 1
    [show hide columns] => 1
    [view schema] => 1
    [add_feedburner_feed] => 1
    [subscribed] => 1
)
~~~

and 

~~~
print_r(Dataface_PermissionsTool::READ_ONLY());
~~~

results in:

~~~
Array
(
    [view in rss] => 1
    [view] => 1
    [link] => 1
    [list] => 1
    [calendar] => 1
    [view xml] => 0
    [show all] => 1
    [find] => 1
    [navigate] => 1
    [ajax_load] => 1
    [find_list] => 1
    [find_multi_table] => 1
    [rss] => 1
    [export_csv] => 0
    [export_xml] => 0
    [export_json] => 1
    [view related records] => 1
    [related records feed] => 1
    [expandable] => 1
    [find related records] => 1
    [link related records] => 1
    [show hide columns] => 1
    [add_feedburner_feed] => 0
    [subscribed] => 1
)
~~~

and

~~~
print_r(Dataface_PermissionsTool::NO_ACCESS());
~~~

results in

~~~

Array
(
    [view] => 0
    [link] => 0
    [view in rss] => 0
    [list] => 0
    [calendar] => 0
    [edit] => 0
    [new] => 0
    [select_rows] => 0
    [post] => 0
    [copy] => 0
    [update_set] => 0
    [update_selected] => 0
    [add new related record] => 0
    [add existing related record] => 0
    [delete] => 0
    [delete selected] => 0
    [delete found] => 0
    [show all] => 0
    [remove related record] => 0
    [delete related record] => 0
    [view related records] => 0
    [view related records override] => 0
    [related records feed] => 0
    [update related records] => 0
    [find related records] => 0
    [edit related records] => 0
    [link related records] => 0
    [find] => 0
    [import] => 0
    [export_csv] => 0
    [export_xml] => 0
    [export_json] => 0
    [translate] => 0
    [history] => 0
    [edit_history] => 0
    [navigate] => 0
    [reorder_related_records] => 0
    [ajax_save] => 0
    [ajax_load] => 0
    [ajax_form] => 0
    [find_list] => 0
    [find_multi_table] => 0
    [register] => 0
    [rss] => 0
    [xml_view] => 0
    [view xml] => 0
    [manage_output_cache] => 0
    [clear views] => 0
    [manage_migrate] => 0
    [manage] => 0
    [manage_build_index] => 0
    [install] => 0
    [expandable] => 0
    [show hide columns] => 0
    [view schema] => 0
    [add_feedburner_feed] => 0
    [subscribed] => 0
)
~~~

These methods are just returning an associative array that indicates whether permissions are granted or disallowed in a particular context.  All of these permissions are defined in the [permissions.ini file](../permissions.ini).  In addition, modules can define their own permissions in their own `permissions.ini files`, and so can applications.  

###Permission Groups / getRolePermissions()

If you look inside the [permissions.ini file](../permissions.ini), you'll notice that all of the individual permission names are defined at the beginning of the file as name/description pairs.  But the end of the file consists of named sections whose properties assign `0`-`1` values to the individual permissions.  These sections are permission sets (sometimes called *roles*).  For example, this section of the permissions.ini file defines the "READ ONLY" permission set, which is actually used by the `Dataface_PermissionsTool::READ_ONLY()` method as a source for its permission set:

~~~
[READ ONLY]
	view in rss=1
	view = 1
	link = 1
	list = 1
	calendar = 1
	view xml = 1
	show all = 1
	find = 1
	navigate = 1
	ajax_load = 1
	find_list = 1
	find_multi_table = 1
	rss = 1
	export_csv = 1
	export_xml = 1
	export_json = 1
	view related records=1
	related records feed=1
	expandable=1
	find related records=1
	link related records=1
	show hide columns = 1
~~~

This section explicitly sets which permissions are granted as part of the `READ ONLY` permission set.  We can obtain this permission set from code using the `Dataface_PermissionsTool::getRolePermissions()` method:

~~~
return Dataface_PermissionsTool::getRolePermissions('READ ONLY');
~~~

##Xataface Core Permissions

Xataface defines the following core permissions:

| Permission Name | Description | Included in  | Excluded in |
|---|---|---|
| `view` | View record contents | `READ BASIC`, `READ ONLY` |  |
| `link` | Link to given record. | `READ BASIC`, `READ ONLY` |  |
| `view in rss` | View record as part of RSS feed | `READ ONLY` |  |
| `list` | Access to the *list* view | `READ BASIC`, `READ ONLY` | |
| `calendar` | Access to the *calendar* action | `READ BASIC`, `READ ONLY` | |
| `edit` | Edit access to a record or field. | `EDIT BASIC`, `EDIT` |  |
| `new` | Access to create new record, or edit field on new record form. | `EDIT BASIC`, `EDIT` | `OWNER` |
| `select_rows` | Select rows in the list view | `EDIT BASIC`, `EDIT` | |
| `copy` | Copy a record | `EDIT BASIC`, `EDIT` | |
| `update_set` | Access to the update result set action | `EDIT BASIC`, `EDIT` | |
| `update_selected` | Access to the update selected records action | `EDIT BASIC`, `EDIT` | |
| `add new related record` | Add a new related record to a relationship | `EDIT`, `EDIT BASIC`, `USER` |  |
| `add existing related record` | Add an existing record to a relationship | `EDIT`, `EDIT BASIC` |  |
| `delete` | Delete a record | `DELETE`, `DELETE BASIC` | |
| `delete selected` | Delete selected records in list view | `DELETE`, `DELETE BASIC` | |
| `delete found` | Delete the full found set. | `DELETE`, `DELETE BASIC` | |
| `show all` | Show all records in the found set. | `READ ONLY`, `READ BASIC` | |
| `remove related record` | Remove a record from a relationship. | `EDIT BASIC`, `EDIT`| |
| `delete related record` | Remove a record from a realtionship, and delete the source record. |  |  |
| `view related records` | View the records of a relationship | `READ BASIC`, `READ ONLY` | |
| `view related records override` | An override permission for related records to allow viewing of a record when through the lens of a relationship even if the record itself does not allow the `view` permission. | |
| `related records feed` | Access to the RSS feed for a relationship. | `READ ONLY` | |
| `update related records` | Access to the update related records action (update multiple at once) | `EDIT BASIC`, `EDIT` | |
| `find related records` | Access to find related records | `READ BASIC`, `READ ONLY` | |
| `edit related records` | Access to edit related records.  Overrides permissions of the source record. | `EDIT BASIC`, `EDIT` | |
| `link related records` | Permission to click link on records in a relationship. | `READ BASIC`, `READ ONLY` |  |
| `find` | Permission to perform the find action. | `READ BASIC`, `READ ONLY` | |
| `import` | Permission to import records. | `EDIT BASIC`, `EDIT` | |

