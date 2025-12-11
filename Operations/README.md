# Operations Documentation

| Operation | Description |
| :--- | :--- |
| [2FA Configuration](2fa.md) | Setup Two-Factor Authentication |
| [Device Management](device-management.md) | Manage devices, approval, isolation |
| [Policy Management](polices.md) | Manage ACLs, groups, and hosts |
| [DNS Management](dns-management.md) | Manage DNS in your organization |
| [Node Gateway](routes.md) | Node Gateway configuration |

## Authentication

You can login to the ZTrust Controller using the following credentials:

Password is value set for `SUPER_ADMIN_PASSWORD` in the `.env` file. Default password is `changeme!`

- Username: `superadmin`
- Password: `changeme!`

![Login Form](./images/login.png)

## Dashboard

User and device overview
![Dashboard](./images/dashboard.png)

Time usage of each user
![Time usage](./images/time_usage.png)

## Device Management

Manage device, approve, isolate, delete device in your organization

![Manage device](./images/manage_device.png)

See [Device Management](device-management.md) for more details

## Peer Connections

Manage peer connections in your organization

![Peer connections](./images/peer_connections.png)

## Network Policy Management

Manage ACL, groups, hosts, etc in your network

![Policy management](./images/policy_management.png)

See [Policy Management](polices.md) for more details

## Posture Checks

Manage posture checks in your organization

![Posture checks](./images/posture_checks.png)

## Routing DNS 

Manage routing and DNS in your organization

![Routing DNS](./images/routing_dns.png)

## Routing Configuration

Manage routing configuration in your organization

![Routing configuration](./images/routing_configuration.png)

## Audit Logs

Audit logs show all actions performed by users and devices in your organization

![Audit logs](./images/audit_logs.png)

## User Management

Manage user accounts and device ownership

![User management](./images/user_management.png)
