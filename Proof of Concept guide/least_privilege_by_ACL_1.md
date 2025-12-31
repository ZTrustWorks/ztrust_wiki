# Nodes Can Access Only Explicitly Authorized Services

## 1. Policy Creation

Go to the `Network Policy` tab → ACLs → Create ACL.

1. Allow devices of users in the `data_entry` group and devices of user `dongtv28` to access the data entry system at 10.110.84.110 on ports 3000, 8000, and 8005.

   **Supported Source/Destination types:** `IP/CIDR`, `Group`, `Host`, `User`
   ```
   Name: Allow Data Entry
   Description: Allow the data entry staff group to access the system at 10.110.84.110 on ports 3000, 8000, and 8005
   Protocol: ALL (TCP, UDP)
   ACL Effective Time Range: None (no access schedule is required)
   Source: User:dongtv28, Group:data_entry
   Destinations: 10.110.84.110:3000, 10.110.84.110:8000, 10.110.84.110:8005
   ```
   
   ![img.png](images/create_acl.png)



3. Click `Save` Changes to save the policy.
4. Click `Commit` Changes. The interface will display the modified policy entries. The administrator must provide a commit reason and then click `Confirm & Submit`.
![img.png](images/comit_acl.png)

   **Note: By default, newly created policies are not automatically enabled.**
   You must click Enabled to ensure the policy is enforced (the propagation process takes approximately 1 minute).
   ![img_1.png](images/enabled_acl.png)

**Result**
- User `dongtv28` and members of the data_entry group can access 10.110.84.110 on ports 3000, 8000, and 8005.
![img.png](images/output_acl.png)
- User `dongtv28` and members of the data_entry group cannot access 10.110.84.110 on port 3001.
![img.png](images/output_acl2.png)
