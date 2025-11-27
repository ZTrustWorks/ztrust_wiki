# Polices Management

## Groups Management

### Create Group

Policy group is used to group policies for easy management

Tab Policy Group -> Create Group
![Click Create Group](./images/create_group_btn.png)

Select user you want to add to group

![Create Group](./images/create_group.png)

Click Create Group

![Create Group](./images/click_create_group.png)

Group `security` is created

![Create Group](./images/created_group.png)

### Edit Group

Click to group name to edit group

### Delete Group

Click to trash icon to delete group

![Delete Group](./images/delete_group.png)

## ACL Management

### Create ACL

Tab ACL -> Create ACL
![Click Create ACL](./images/create_acl_btn.png)

Enter ACL name and description

![Create ACL Form](./images/create_acl_form.png)

You can set time range or schedule for ACL
This feature is useful for setting policies for specific time ranges or schedules (Just In Time)

![ACL JIT](./images/acl_jit.png)

Select source and destination for ACL and click `Add`

The type source and destination can be :

- All: Allow all source or destination
- IP/CIDR: Allow specific IP or CIDR
- Group: Allow specific group
- User: Allow specific user
- Host: Allow specific host

![Select acl group](./images/select_acl_group.png)

Select destination and config port for ACL
![Selct acl dest](./images/select_acl_dest.png)

Click `Create ACL` for create ACL

Now ACL is created, we can click to enable the ACL, then commit to apply the ACL
![Enable ACL](./images/enable_acl.png)

Click `Commit Changes` to apply the ACL
![Commit ACL](./images/commit_acl.png)

Enter description for commit change and click `Confirm & Submit` to apply the ACL

![Commit Changes](./images/commit_changes.png)
