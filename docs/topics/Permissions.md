# Permissions

Permissions in Discord are a way to limit and grant certain abilities to users. A set of base permissions can be configured at the guild level for different roles. When these roles are attached to users, they grant or revoke specific privileges within the guild. Along with the guild-level permissions, Discord also supports permission overwrites that can be assigned to individual guild roles or guild members on a per-channel basis.

Permissions are stored within a 53-bit integer and are calculated using bitwise operations. The total permissions integer can be determined by ORing together each individual value. Additional logic is required when permission overwrites are involved; this is further explained below. For more information about bitwise operations and flags, see [this page](https://en.wikipedia.org/wiki/Bit_field).

Below is a table of all current permissions, their integer values in hexadecimal, brief descriptions of the privileges that they grant, and the channel type they apply to, if applicable.

###### Bitwise Permission Flags

| Permission | Value | Description | Channel Type |
|------------|-------|-------------|--------------|
| CREATE\_INSTANT\_INVITE | `0x00000001` | Allows creation of instant invites | T, V |
| KICK\_MEMBERS * | `0x00000002` | Allows kicking members | |
| BAN\_MEMBERS * | `0x00000004` | Allows banning members | |
| ADMINISTRATOR * | `0x00000008` | Allows all permissions and bypasses channel permission overwrites | |
| MANAGE\_CHANNELS * | `0x00000010` | Allows management and editing of channels | T, V |
| MANAGE\_GUILD * | `0x00000020` | Allows management and editing of the guild | |
| ADD\_REACTIONS | `0x00000040` | Allows for the addition of reactions to messages | T |
| VIEW\_AUDIT\_LOG | `0x00000080` | Allows for viewing of audit logs | |
| VIEW\_CHANNEL | `0x00000400` | Allows guild members to view a channel, which includes reading messages in text channels | T, V |
| SEND\_MESSAGES | `0x00000800` | Allows for sending messages in a channel | T |
| SEND\_TTS\_MESSAGES | `0x00001000` | Allows for sending of `/tts` messages | T |
| MANAGE\_MESSAGES *  | `0x00002000` | Allows for deletion of other users messages | T |
| EMBED\_LINKS | `0x00004000` | Links sent by users with this permission will be auto-embedded | T |
| ATTACH\_FILES | `0x00008000` | Allows for uploading images and files | T |
| READ\_MESSAGE\_HISTORY | `0x00010000` | Allows for reading of message history | T |
| MENTION\_EVERYONE | `0x00020000` | Allows for using the `@everyone` tag to notify all users in a channel, and the `@here` tag to notify all online users in a channel | T |
| USE\_EXTERNAL\_EMOJIS | `0x00040000` | Allows the usage of custom emojis from other servers | T |
| CONNECT | `0x00100000` | Allows for joining of a voice channel | V |
| SPEAK | `0x00200000` | Allows for speaking in a voice channel | V |
| MUTE\_MEMBERS | `0x00400000` | Allows for muting members in a voice channel | V |
| DEAFEN\_MEMBERS | `0x00800000` | Allows for deafening of members in a voice channel | V |
| MOVE\_MEMBERS | `0x01000000` | Allows for moving of members between voice channels | V |
| USE\_VAD | `0x02000000` | Allows for using voice-activity-detection in a voice channel | V |
| PRIORITY\_SPEAKER | `0x00000100` | Allows for using priority speaker in a voice channel | V |
| CHANGE\_NICKNAME | `0x04000000` | Allows for modification of own nickname | |
| MANAGE\_NICKNAMES | `0x08000000` | Allows for modification of other users nicknames | |
| MANAGE\_ROLES * | `0x10000000` | Allows management and editing of roles | T, V |
| MANAGE\_WEBHOOKS * | `0x20000000` | Allows management and editing of webhooks | T, V |
| MANAGE\_EMOJIS * | `0x40000000` | Allows management and editing of emojis | | |

**\* These permissions require the owner account to use [two-factor authentication](#DOCS_TOPICS_OAUTH2/twofactor-authentication-requirement) when used on a guild that has server-wide 2FA enabled.**

Note that these internal permission names may be referred to differently by the Discord client. For example, "Manage Permissions" refers to MANAGE_ROLES, "Read Messages" refers to VIEW_CHANNEL, and "Use Voice Activity" refers to USE_VAD.

## Permission Hierarchy

How permissions apply may at first seem intuitive, but there are some hidden restrictions that prevent bots from performing certain inappropriate actions based on a bot's highest role compared to its target's highest role. A bot's or user's highest role is its role that has the greatest position value in the guild, with the default @everyone role starting at 0. Permissions follow a hierarchy with the following rules:

* A bot can grant roles to other users that are of a lower position than their highest role.
* A bot can edit roles of a lower position than their highest role, but they can only grant permissions they have to those roles.
* Bots can only sort roles lower than their highest role.
* Bots can only kick/ban users whose highest role is lower than the bot's highest role.

Otherwise, permissions do not obey the role hierarchy. For example, a user has two roles: A and B. A denies the `VIEW_CHANNEL` permission on a #coolstuff channel. B allows the `VIEW_CHANNEL` permission on the same #coolstuff channel. The user would ultimately be able to view the #coolstuff channel, regardless of the role positions.

## Permission Overwrites

Certain permissions can be applied to roles or directly to members on a channel-level by using permission overwrites. Applicable permissions are indicated by a **T** for text-channels or **V** for voice-channels in the table above.

When using overwrites, there are cases where permission collisions could occur for a user; that is to say, the user may have certain overwrites with permissions that contradict each other or their guild-level role permissions. With this in mind, permissions are applied to users in the following hierarchy:

1. Base permissions given to @everyone are applied at a guild level
2. Permissions allowed to a user by their roles are applied at a guild level
3. Overwrites that deny permissions for @everyone are applied at a channel level
4. Overwrites that allow permissions for @everyone are applied at a channel level
5. Overwrites that deny permissions for specific roles are applied at a channel level
6. Overwrites that allow permissions for specific roles are applied at a channel level
7. Member-specific overwrites that deny permissions are applied at a channel level
8. Member-specific overwrites that allow permissions are applied at a channel level

The follow pseudocode demonstrates this process programmatically:

```python
def compute_base_permissions(member, guild):
    if guild.is_owner(member):
        return ALL

    role_everyone = guild.get_role(guild.id)  # get @everyone role
    permissions = role_everyone.permissions

    for role in member.roles:
        permissions |= role.permissions

    if permissions & ADMINISTRATOR == ADMINISTRATOR:
        return ALL

    return permissions

def compute_overwrites(base_permissions, member, channel):
    # ADMINISTRATOR overrides any potential permission overwrites, so there is nothing to do here.
    if base_permissions & ADMINISTRATOR == ADMINISTRATOR:
        return ALL

    permissions = base_permissions
    overwrite_everyone = overwrites.get(channel.guild_id)  # Find (@everyone) role overwrite and apply it.
    if overwrite_everyone:
        permissions &= ~overwrite_everyone.deny
        permissions |= overwrite_everyone.allow

    # Apply role specific overwrites.
    overwrites = channel.permission_overwrites
    allow = NONE
    deny = NONE
    for role_id in member.roles:
        overwrite_role = overwrites.get(role_id)
        if overwrite_role:
            allow |= overwrite_role.allow
            deny |= overwrite_role.deny

    permissions &= ~deny
    permissions |= allow

    # Apply member specific overwrite if it exist.
    overwrite_member = overwrites.get(member.user_id)
    if overwrite_member:
        permissions &= ~overwrite_member.deny
        permissions |= overwrite_member.allow

    return permissions

def compute_permissions(member, channel):
    base_permissions = compute_base_permissions(member, channel.guild)
    return compute_overwrites(base_permissions, member, channel)
```

## Implicit Permissions

Permissions in Discord are sometimes implicitly denied or allowed based on logical use. The two main cases are `VIEW_CHANNEL` and `SEND_MESSAGES` for text channels. Denying a user or a role `VIEW_CHANNEL` on a channel implicitly denies other permissions on the channel. Though permissions like `SEND_MESSAGES` are not explicitly denied for the user, they are ignored because the user cannot read messages in the channel.

Denying `SEND_MESSAGES` implicitly denies `MENTION_EVERYONE`, `SEND_TTS_MESSAGES`, `ATTACH_FILES`, and `EMBED_LINKS`. Again, they are not explicitly denied when doing permissions calculations, but they are ignored because the user cannot do the base action of sending messages.

There may be other cases in which certain permissions implicitly deny or allow other permissions. In all cases, it is based on logical conclusions about how a user with certain permissions should or should not interact with Discord.

## Permission Syncing

Permissions with regards to categories and channels within categories are a bit tricky. Rather than inheritance, permissions are calculated by means of what we call Permission Syncing. If a child channel has the same permissions and overwrites (or lack thereof) as its parent category, the channel is considered "synced" to the category. Any further changes to a **parent category** will be reflected in its synced child channels. Any further changes to a **child channel** will cause it to become de-synced from its parent category, and its permissions will no longer change with changes to its parent category.

### Role Object

Roles represent a set of permissions attached to a group of users. Roles have unique names, colors, and can be "pinned" to the side bar, causing their members to be listed separately. Roles are unique per guild, and can have separate permission profiles for the global context (guild) and channel context.

###### Role Structure

| Field | Type | Description |
|-------|------|-------------|
| id | snowflake | role id |
| name | string | role name |
| color | integer | integer representation of hexadecimal color code |
| hoist | bool | if this role is pinned in the user listing |
| position | integer | position of this role |
| permissions | integer | permission bit set |
| managed | bool | whether this role is managed by an integration |
| mentionable | bool | whether this role is mentionable |

Roles without colors (`color == 0`) do not count towards the final computed color in the user list.

###### Example Role

```json
{
	"id": "41771983423143936",
	"name": "WE DEM BOYZZ!!!!!!",
	"color": 3447003,
	"hoist": true,
	"position": 1,
	"permissions": 66321471,
	"managed": false,
	"mentionable": false
}
```
