V1 JDK / Tomcat Rollout
-----------------------

**Tables involved :**
AgentRankView
Rollout (TYPE denotes JDK or Tomcat)
RolloutSchedule
RolloutResource 
RolloutDetail

**Schedule [Manager]:**
RolloutManager (RolloutTask.class)

**Provisioning -> Component Rollout**

**Procedure :**

#. Add JDK binary to all DEs
#. Patch cacerts 
#. Mail SD with a rollout schedule or ask them for one
#. Add Rollout -> Fill out the schedule that was agreed upon and submit
#. SD will then notify App teams via SD Users on Connect and "Start Rollout" for all resources. There's also a provision to roll the binary out to specific resources and also to exclude specific resources from a global rollout (this can come in handy if a particular app alone has some issue with the new version that's being rolled out)
#. The schedule, RolloutManager, runs every 5 mins. The task iterates over all DEs. If current time is found to be less than the scheduled time for the first time, RolloutDetail will be populated with the involved ASGs / App Servers [?](not sure if they are ASG-wise or AppServer-wise rows) and default values. ACTION_ID will be null for all rows the first time. From the second iteration, JDK Upgrade API will be invoked in the respective DE for 10 Apps at a time. This limit of 10 Apps per iteration is imposed so as to not overload the agents. Once the API is invoked, its ACTION_ID would be persisted and the status will be polled and persisted in the consecutive iterations. The status can be viewed by clicking "Show Status". Failed entities can be viewed by applying "Action Status" filter, by choosing "Others"
#. Once action is invoked for all resources, It's important to mark the rollout as completed / aborted. This will avoid repeated retries and therefore spare the agent log files. 

**Tomcat Upgrade, Agent (Not in use right now)-**
Just like JDK upgrade, there's an option to upgrade Tomcat in App Servers. 

**Workflow :**

#. Tomcat binary would be transferred to the App Server
#. App server process will be killed
#. A copy of all WARs within webapps, conf/server.xml.orig, conf/logging.properties, conf/*.keystore, bin/setenv.sh will be taken and then the existing tomcat directory will be discarded. 
#. The transferred binary will be extracted within AdventNet/Sas/ and the App Server will be restarted. 
#. In a normal App Server Upgrade (ASU), the entire AdventNet directory would be updated. So, the previously upgraded Tomcat version would get overwritten. To work around this, for all consecutive ASUs that follow a Tomcat Upgrade, we would copy the Tomcat binary along with app TAR to the remote App Server, extract TAR, overwrite tomcat directory while retaining the above stated files and then restart. For this, data from the table AppGroupTemplate will be used.
#. Say there’s an app that has customized tomcat/conf/context.xml. Now, this is a file that we do not retain while overwriting Tomcat. When there’s an emergency and if they require us to allow them to upgrade just their TAR and not have their Tomcat overwritten, there’s an option to disable the same -> Manager - Provisioning -> E/D Tomcat Upgrade
#. Once this is disabled, tomcat that comes bundled with TAR will be used. We may either enable the same again or do a Tomcat Upgrade for the App in order to enable the overwriting once again.


v2 Nginx rollout
-----------------
Mostly similar to the above but works with different code and a different UI. A part of data is stored in tables involved in v3 (Library and LibraryVersion ?)

**Manager -> LB -> L7 Version, L7 Channels, L7 Rollout**

**L7 Version -** 
Not used, generally. Not sure if it’s in working condition.
Zorro usually uploads Nginx binaries via agent and then use Manager just to initiate rollouts.

**L7 Channels -**

#. Data is persisted in manager DB.
#. A channel may comprise one or more L7 clusters from one or more DEs. 

The original idea was to support something like the following - Maintain 3 channels with all low priority clusters of all DEs in a channel, all medium priority clusters of all DEs in another and all high priority clusters of all DEs in the other. This would make just 3 neatly categorized channels. However, there's a lot of channels right now and if Zorro is comfortable managing them, we'll let it be. (I’m not sure if there’s a wiki with the procedure. Santha will be able to find the email that we had sent Zorro. Worth documenting.)

**L7 Rollout -**

#. Add Rollout - We add rollout with the version label and then there’s a field that says “Interval in mins”. This represents the time gap between each upgrade API invocation.
#. Schedule involved - L7RolloutManager
#. The schedule is similar to the above except only one *node* in each DE will be upgraded in a single run. Upgrade API that’s invoked at the agent will not upgrade the entire cluster but just one of the nodes in the cluster. When the upgrade is found to have failed, the rollout would get paused and will not proceed until the issue is fixed and there’s a successful upgrade of the same. In most cases, the issue is usually with the binary and Zorro fixes it. They are then supposed to “Retry failed upgrade” from the L7 Rollout page to reinitiate the upgrade of the failed node. If the fix works and the upgrade turns out to be successful, the rollout resumes by default. *[This has been working quite smoothly and there weren’t many issues as such. However, we need to enable a similar rollout workflow for ZCP clusters as well - this is unaddressed]*
#. We distinguish main and DR with the help of data from AgentPeer. This might need updates after every switch. **This cannot be done via UI and is manually fed through data migration.**


v3 MQ / SasProv
---------------
**Tables involved [?](May be incorrect. Correct if wrong.) :**

LibraryLevel
Library
CmtoolsLibrary
LibVersion
LibVersionToDEs
LibraryChannel
AppSubscribers
ASGSubscribers
LibRollout
….

**Library -**
Add Library -> Provide a name without space and a handler class. In case we add “com.zoho.upgrade.SasProvisioningLibrary”, it will get persisted as “com.zoho.upgrade.web.SasProvisioningLibrary” in manager and “com.zoho.upgrade.SasProvisioningLibrary” in agent. There is an additional “.web.” in manager alone.  *Additionally, it’s required that the configured class and that that extends “Library” is deployed even if the library needs no customization.*

**LibraryVersion -**

#. Choose Library from the top-left dropdown and use “Add version”. 
#. This option does an HTTP upload to the chosen Agents. 
#. Binaries are persisted in the respective Agent's disk within - ~/binaries/libraries/<LibraryName>/<VersionLabel>/<libraryname.zip> (~/binaries/libraries/SasProvisioning/M21_535/sasprovisioning.zip)
#. SAS_Provisioning.zip usually is around 2GB big and it takes a long time to upload even with intranet. Even when the request times out in manager's end and the page turns unresponsive, uploads to agents usually continue and would get done after a while. But with slow connections and over VPN, uploads to manager itself seem to fail frequently. **This is a showstopper and needs an alternate really soon.**
#. Once binary is uploaded to agents, the version can be mapped to one or more channels. Mapping different versions to different channels is tedious right now because we’re supposed to change the agent dropdown every time and then map one by one. Because we roll out the latest SasProv version to all Apps anyway, we can use “Map version to all channels”. This would map the particular version to all channels in all DEs. 
#. **Set DBD -** We can then set a deploy-before-date for the version. This is the deadline that we impose on SD team to get the version updated in the respective channel & DE. **We usually send a mail to the SD team after we set a DBD for them to schedule and start the rollout. This mail notification can be automated.**

**Library Channel -**

#. There can be multiple channels in a DE. Channel details are persisted in Agent’s end and so a channel cannot have subscribers across different agents. 
#. Subscribers can be Apps or ASGs or any entity for that matter (overriddable). MQ and SasProv channel subscribers are populated with a schedule. This schedule basically fetches SaSLite channel data from CMTools channel related tables and populates in library related tables. Saslite's channel1, channel2 and channel3 apps get mapped to SasProv's channel1, channel2 and channel3 respectively. Apps that are not mapped to any SaSLite channel would be mapped to SasProv's channel0.

**Rollout : Agent -**

#. There’ll be rollout entries added in Agents once a DBD is set for a version. A schedule needs to be made by SD team such that the scheduled time falls within the set DBD.
#. Schedule and start rollout happen in Agents. Rollout progress can be monitored in the respective Agent. 
#. Once “Start Rollout” is done, upgrades would be carried out for 5(?) apps at a time. Status can be monitored by SD in Agent itself.
#. **Autopilot mode -** This can be enabled library-wise. If this is enabled for a library in an agent, scheduling the rollout by SD team can be skipped; the rollout would get started 1hr just before the DBD on its own. 

**So, when automatic notification is worked out and autopilot mode is enabled for a library, there needn't be SD team's involvement at all.**

SasProv Rollout - Things to do : 
--------------------------------
If somebody in the team requests us for a rollout, these are the steps to be followed :
  #. Download the corresponding version of SasProvisioning. 
  #. Upload the ZIP from the Library Version page. 
  
     #. Intranet way : Go to Library Version page and "Add Version" and upload the ZIP. 
     #. VPN : Upload a dummy.zip with no content to required DEs. This will take care of making the necessary DB entries. Now, rename SAS_Provisioning.zip to sasprovisioning.zip and scp to all agents, under ~/binaries/libraries/SasProvisioning/M20_XXX/ [Fix this ASAP]
     
  #. Map the version to all channels from the same Library Version page. 
  #. Tell them the version is made available in all DEs and that they may set a DBD 
  #. They will set DBD and mail SD team to take over. [This mail notification may be automated]
[If 2 is fixed, we needn't get involved at all]

SasProv Upgrades in Agent 
-------------------------

There is an option to upgrade SasProvisioning via manager, open to developers. There's also another option in agent from where SasProvisioning can be upgraded to multiple Apps in one shot. 

**Scheduler :**
SasProvisioningJob

**Workflow :**

#. SasProvisioning binaries would be utilised from the path in LibraryVersion table. binaries/libraries/SasProvisioning/M21_XXX/sasprovisioning.zip.
#. There are some pre-allowed configuration files where we allow customization, viz. mysql/mysql.cnf, mysql/mysql_preinstall.sh, mysql/mysql_postinstall.sh, postgres/postgres_preinstall.sh, postgres/postgres_postinstall.sh, fos/fos.cfg. 
#. The custom files are stored in Agent's disk at ~/binaries/applications/<AppName>/<provisioning>/... For example, ~/binaries/applications/Crm/provisioning/mysql/mysql.cnf
#. In an intention to preserve edit history, we additionally persist the file contents in a table. We do not have a UI for edit history. So, in essence, this DB persistence can be skipped. **There were a few instances where the content was huge and startGrid failed while trying to persist in the DB.** File persistence is skipped for startGrid alone.
#. Customization UI is exposed to SD team and our team alone. The idea is to expose it to app developers after placing enough checks. 
#. Customization options :

   #. Add file : File name (mysql/mysql.cnf), file content and DEs are supposed to be fed. In case the file is a binary and not plain text, the user can make use of the upload option.
   #. Clone : Say, for an application, mysql.cnf is customized in CT1 alone and the developer plans to expand the same to a few other DE's where there's no custom mysql.cnf yet. He can use this option to copy CT1's version of mysql.cnf in the said DE's.
   #. Link : Say, for an application, one version of mysql.cnf exists in CT1 alone, another version of mysql.cnf exists in all US4 and the developer wishes to have the same configuration in CT1 as that of US4, he may use link. This is usually done to homogenize configurations across DEs. So, clone will introduce a new       custom file in a DE and link with help synchronize them.
   #. Some data is persisted and retreived from Manager DB although Agent has the master copy. We aimed to have the master copy in DBS but was not done. **After each such action, a SasProvisioning upgrade has to be done in order for the files to reflect in the respective App's backup cluster.
  
#. For ZCP agents, during Apps' startGrid, SasProv ZIP is transferred from a default location in ZAC agents. (~/binaries/provisioning/SAS_Provisioning.zip?) This ZIP is generally placed when the agent is deployed for the first time. When the ZIP goes outdated after a while, we're supposed to replace it with an updated version. (We could no track down who's supposed to place the ZIP there, so it fell upon us.) This needs a simple fix; we can tinker the startGrid code to query LibraryVersion table for the location of the latest SasProv ZIP and use it, instead of the default location. (Mind the SAS_Provisioning.zip and sasprovisioning.zip here.)
#. For including custom files during startGrid, we have handed out a list of things to do, to SD team. Must be there in the SasProv wiki. 
#. New ZAC deployment, we tell the SD to copy - ~/backup/applications/<AppName>/provisioning/* for all Apps from the blueprint DE, while retaining the directory structure. startGrid code (agent) copies these files to the backup cluster for them to be involved. 
#. ( https://writer.zoho.com/writer/open/dmgmj988658f3ae2743ecb81c14233fba650c - Not sure if the doc is still accessible; this has some more details.)


v4 (Involving DBS)
-------------------
Santhosh Manikandan must have a rough idea. 


Manager Upgrades - CDep
-----------------------
A petty thing that may need to be fixed - In case an upgrade involving a metadata migration fails, right now there's no way we can fix and rerun the upgrade without a manual intervention. Deleting the last / latest entry from SelfUpgradeAction table would reset the previous state and a successive retry usually works. This glitch needs to be worked around.

