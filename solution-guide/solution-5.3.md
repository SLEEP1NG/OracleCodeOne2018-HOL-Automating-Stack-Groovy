# 5.3 Setup Nexus access

## Task

Try writing the Groovy code to set up the Sea Lion’s access. Remember that they get access to the Osprey code in addition to their own

## Solution

````groovy
import org.sonatype.nexus.script.plugin.*
import org.sonatype.nexus.selector.*
import org.sonatype.nexus.security.*
import org.sonatype.nexus.common.app.*;
import org.sonatype.nexus.common.entity.*
import org.sonatype.nexus.security.authz.*
import org.sonatype.nexus.security.user.*

// much of this code is based off of https://gist.github.com/nblair/1a0e05713c3edb7e5360c2b0222c7623

// use container.lookup to fetch internal APIs we need to use
selectorManager = container.lookup(SelectorManager.class.name)
securitySystem = container.lookup(SecuritySystem.class.name)
authorizationManager = securitySystem.getAuthorizationManager('default')

// -------------------------------------------------------------------------

def addOrReplaceRole(role) {
   def originalRoles = authorizationManager.listRoles()
   if (originalRoles.grep { it.roleId == role.roleId }) {
	   authorizationManager.deleteRole(role.roleId)
   }
   authorizationManager.addRole(role)
}

def addOrReplacePrivilege(privilege) {
	def originalPrivileges = authorizationManager.listPrivileges()
	if (originalPrivileges.grep { it.id == privilege.id }) {
		authorizationManager.deletePrivilege(privilege.id)
	}
	authorizationManager.addPrivilege(privilege)
 }

// -------------------------------------------------------------------------
// anonymous user cannot see osprey or sea lion code
def limitedAnonRole = new org.sonatype.nexus.security.role.Role(
	roleId: "limited-anon",
	source: "Nexus",
	name: "Limited Anonymous",
	description: "Browse access without access to all group ids",
	readOnly: false,
	privileges: [ 'nx-search-read', 'nx-repository-view-maven2-*-read' ],
	roles: []
)
addOrReplaceRole(limitedAnonRole)

def anonUser = securitySystem.getUser('anonymous')


// -------------------------------------------------------------------------

// create content selector (if not already present)
def ospreySelectorConfig = new SelectorConfiguration(
	name: 'osprey-selector',
	type: 'csel',
	description: 'sea-lion packages',
	attributes: ['expression': 'format == "maven2" and coordinate.groupId =^ "net.selikoff.oraclecodeone.groovy.osprey"']
)
if (selectorManager.browse().find { it -> it.name == ospreySelectorConfig.name } == null) {
  selectorManager.create(ospreySelectorConfig)
}

// -------------------------------------------------------------------------

// create privileges for release and snapshot repositories
def ospreyReleaseProperties = ['contentSelector' : ospreySelectorConfig.name, 
   'repository' : 'custom-releases', 'actions' : 'browse,read,edit']
def ospreyReleasePrivilege = new org.sonatype.nexus.security.privilege.Privilege(
	id: "osprey-release-priv",
	version: '',
	name: "osprey-release-priv",
	description: "Content Selector Release privilege",
	type: "repository-content-selector",
	properties: ospreyReleaseProperties
)

def ospreySnapshotProperties = ['contentSelector' : 'osprey-selector',
	'repository' : 'custom-snapshots', 'actions' : 'browse,read,edit']
def ospreySnapshotPrivilege = new org.sonatype.nexus.security.privilege.Privilege(
	id: "osprey-snapshot-priv",
	version: '',
	name: "osprey-snapshot-priv",
	description: "Content Selector Snapshot privilege",
	type: "repository-content-selector",
	properties: ospreySnapshotProperties
)
addOrReplacePrivilege(ospreyReleasePrivilege)
addOrReplacePrivilege(ospreySnapshotPrivilege)

// -------------------------------------------------------------------------
// create a role with the snapshot and release privileges
def ospreyRole = new org.sonatype.nexus.security.role.Role(
	roleId: "osprey-role",
	source: "Nexus",
	name: "osprey-role",
	description: "Osprey Role",
	readOnly: false,
	privileges: [ ospreySnapshotPrivilege.id, ospreyReleasePrivilege.id ],
	roles: []
)
addOrReplaceRole(ospreyRole)

// -------------------------------------------------------------------------

// add a local user account with the role
if (! securitySystem.searchUsers(new UserSearchCriteria('olivia')))
  security.addUser("olivia", "Olivia", "O", "olivia@none.com", true, "olivia", [ 'osprey-role', 'limited-anon' ])
if (! securitySystem.searchUsers(new UserSearchCriteria('owen')))
  security.addUser("owen", "Owen", "O", "owen@none.com", true, "owen", [ 'osprey-role', 'limited-anon' ])

  
  if (! securitySystem.searchUsers(new UserSearchCriteria('owen')))
	  security.addUser("duke", "Duke", "O", "owen@none.com", true, "duke", [ 'osprey-role', 'limited-anon' ])

def seaLionSelectorConfig = new SelectorConfiguration(
name: 'sea-lion-selector',
type: 'csel',
description: 'sea-lion packages',
attributes: ['expression': 'format == "maven2" and coordinate.groupId =^ "net.selikoff.oraclecodeone.groovy.sea-lion"']
)
if (selectorManager.browse().find { it -> it.name == seaLionSelectorConfig.name } == null) {
  selectorManager.create(seaLionSelectorConfig)
}

def seaLionReleaseProperties = ['contentSelector' : seaLionSelectorConfig.name,
'repository' : 'custom-releases', 'actions' : 'browse,read,edit']
def seaLionReleasePrivilege = new org.sonatype.nexus.security.privilege.Privilege(
 id: "sea-lion-release-priv",
 version: '',
 name: "sea-lion-release-priv",
 description: "Content Selector Release privilege",
 type: "repository-content-selector",
 properties: seaLionReleaseProperties
)

def seaLionSnapshotProperties = ['contentSelector' : 'sea-lion-selector',
 'repository' : 'custom-snapshots', 'actions' : 'browse,read,edit']
def seaLionSnapshotPrivilege = new org.sonatype.nexus.security.privilege.Privilege(
id: "sea-lion-snapshot-priv",
 version: '',
 name: "sea-lion-snapshot-priv",
 description: "Content Selector Snapshot privilege",
 type: "repository-content-selector",
 properties: seaLionSnapshotProperties
)
addOrReplacePrivilege(seaLionReleasePrivilege)
addOrReplacePrivilege(seaLionSnapshotPrivilege)

def seaLionRole = new org.sonatype.nexus.security.role.Role(
roleId: "sea-lion-role",
source: "Nexus",
name: "sea-lion-role",
description: "Sea Lion Role",
readOnly: false,
privileges: [ seaLionSnapshotPrivilege.id, seaLionReleasePrivilege.id, ospreySnapshotPrivilege.id, ospreyReleasePrivilege.id ],
roles: []
)
addOrReplaceRole(seaLionRole)

if (! securitySystem.searchUsers(new UserSearchCriteria('sam')))
   security.addUser("sam", "Sam", "S", "sam@none.com", true, "sam", [ 'sea-lion-role', 'limited-anon' ])
if (! securitySystem.searchUsers(new UserSearchCriteria('sophia')))
   security.addUser("sophia", "Sophia", "S", "sophia@none.com", true, "sophia", [ 'sea-lion-role', 'limited-anon' ])
````