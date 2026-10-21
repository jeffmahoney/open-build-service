# 06a — Legacy Backend Route Catalog

Generated from the BSDispatch tables at commit db93ae665b. Syntax: `METHOD:/path/$positional key:type? ...` where `?` = optional, `*` = repeatable, `cmd=x` selects the handler, `!-` = no authorization class required. Handler names refer to the Perl functions; see 01 for semantics.

## bs_srcserver (179 routes)

| Route | Perl handler |
|---|---|
| `/` | `hello` |
| `!rw :` | `undef` |
| `!- GET:` | `undef` |
| `!- HEAD:` | `undef` |
| `POST:/source cmd=orderkiwirepos` | `orderkiwirepos` |
| `POST:/source cmd=triggerscmsync scmrepository:? scmbranch:? isdefaultbranch:?` | `triggerscmsync` |
| `POST:/source cmd: *:*` | `unknowncmd` |
| `/source deleted:bool? project*` | `getprojectlist` |
| `POST:/source/$project cmd=createkey user:? comment:?` | `createkey` |
| `POST:/source/$project cmd=extendkey user:? comment:? days:num?` | `extendkey` |
| `POST:/source/$project cmd=recreatesslcert user:? comment:?` | `recreatesslcert` |
| `POST:/source/$project cmd=undelete user:? comment:?` | `undeleteproject` |
| `POST:/source/$project cmd=copy user:? comment:? oproject:project withbinaries:bool? withhistory:bool? makeolder:bool? makeoriginolder:bool? resign:bool? noservice:bool?` | `copyproject` |
| `POST:/source/$project cmd=move oproject:project` | `moveproject` |
| `POST:/source/$project cmd=freezelink user:? comment:? requestid:num?` | `freezeprojectlink` |
| `POST:/source/$project cmd=notifypackagechange` | `notifypackagechange` |
| `POST:/source/$project cmd: *:*` | `unknowncmd` |
| `/source/$project view=info parse:bool? nofilename:bool? repository? arch? package* withchangesmd5:bool? noexpand:bool? withmetamd5:bool?` | `getprojectsourceinfo` |
| `/source/$project deleted:bool? expand:bool? noorigins:bool?` | `getpackagelist` |
| `DELETE:/source/$project user:? comment:? requestid:num?` | `delproject` |
| `/source/$project/_meta rev? withconfig:bool? interconnect:?` | `getproject` |
| `PUT:/source/$project/_meta user:? comment:? requestid:num? lowprio:bool?` | `putproject` |
| `/source/$project/_pubkey rev?` | `getpubkey` |
| `DELETE:/source/$project/_pubkey user:? comment:?` | `deletekey` |
| `/source/$project/_config rev?` | `getprojectconfig` |
| `PUT:/source/$project/_config user:? comment:?` | `putprojectconfig` |
| `DELETE:/source/$project/_config user:? comment:?` | `delprojectconfig` |
| `/source/$project/_history rev? meta:bool? deleted:bool? limit:num? startbefore:num?` | `getpackagehistory` |
| `/source/$project/_keyinfo withsslcert:bool? autoextend:bool? donotcreatecert:bool? signtype:? signflavor:?` | `getkeyinfo` |
| `POST:/source/$project/$package view=info parse:bool? repository? arch? filename?` | `getpackagesourceinfo_post` |
| `POST:/source/$project/$package cmd=diff rev? orev:rev? oproject:project? opackage:package? expand:bool? linkrev? olinkrev:? unified:bool? missingok:bool? meta:bool? file:* filelimit:num? tarlimit:num? view:? withissues:bool? onlyissues:bool? nocache:bool? cacheonly:bool? nodiff:bool?` | `sourcediff` |
| `POST:/source/$project/$package cmd=linkdiff rev? linkrev? unified:bool? file:* filelimit:num? tarlimit:num? view:? withissues:bool? onlyissues:bool? nocache:bool? cacheonly:bool? nodiff:bool?` | `linkdiff` |
| `POST:/source/$project/$package cmd=servicediff rev? unified:bool? file:* filelimit:num? tarlimit:num? view:? withissues:bool? onlyissues:bool? nocache:bool? cacheonly:bool? nodiff:bool?` | `servicediff` |
| `POST:/source/$project/$package cmd=commit rev? user:? comment:? keeplink:bool? repairlink:bool? linkrev? setrev:bool? requestid:num? noservice:bool?` | `sourcecommit` |
| `POST:/source/$project/$package cmd=commitfilelist rev? user:? comment:? keeplink:bool? repairlink:bool? linkrev? setrev:bool? requestid:num? time:num? version:? vrev:? noservice:bool? servicemark:? withvalidate:?` | `sourcecommitfilelist` |
| `POST:/source/$project/$package cmd=commitobsscm srcmd5? servicemark: run:md5 user:? comment:? info:?` | `sourcecommitobsscm` |
| `POST:/source/$project/$package cmd=copy rev? user:? comment:? orev:rev? oproject:project? opackage:package? expand:bool? keeplink:bool? repairlink:bool? linkrev? setrev:linkrev? olinkrev:linkrev? requestid:num? dontupdatesource:bool? noservice:bool? withvrev:bool? withacceptinfo:bool? makeoriginolder:bool? freezelink:bool? vrevbump:num? instantiate:bool?` | `sourcecopy` |
| `POST:/source/$project/$package cmd=collectbuildenv user:? comment:? orev:rev? oproject:project? opackage:package?` | `sourcecollectbuildenv` |
| `POST:/source/$project/$package cmd=branch rev? user:? comment:? orev:rev? oproject:project? opackage:package? olinkrev:linkrev? requestid:num? force:bool? keepcontent:bool? missingok:bool? noservice:bool? withacceptinfo:bool? time:num? extendvrev:bool? simplelink:bool?` | `sourcebranch` |
| `POST:/source/$project/$package cmd=linktobranch rev? user:? comment:? linkrev? noservice:bool?` | `linktobranch` |
| `POST:/source/$project/$package cmd=deleteuploadrev` | `deleteuploadrev` |
| `POST:/source/$project/$package cmd=undelete user:? comment:? time:num?` | `undeletepackage` |
| `POST:/source/$project/$package cmd=runservice user:? comment:?` | `triggerservicerun` |
| `POST:/source/$project/$package cmd=waitservice` | `waitservicerun` |
| `POST:/source/$project/$package cmd=mergeservice user:? comment:?` | `mergeservicerun` |
| `POST:/source/$project/$package cmd=getprojectservices` | `getprojectservices` |
| `POST:/source/$project/$package cmd=notifypackagechange` | `notifypackagechange` |
| `POST:/source/$project/$package cmd=lock user:? comment:?` | `lockpackage` |
| `POST:/source/$project/$package cmd=unlock user:? comment:?` | `unlockpackage` |
| `POST:/source/$project/$package cmd: *:*` | `unknowncmd` |
| `PUT:/source/$project/$package cmd: rev? user:? comment:?` | `sourcecommitfilelist,	# obsolete` |
| `/source/$project/$package view=getmultibuild` | `getmultibuildpackages` |
| `/source/$project/$package view=info rev? linkrev? parse:bool? nofilename:bool? repository? arch? withchangesmd5:bool? noexpand:bool? withmetamd5:bool?` | `getpackagesourceinfo` |
| `/source/$project/$package rev? linkrev? emptylink:bool? deleted:bool? expand:bool? view:? extension:? lastworking:bool? withlinked:bool? meta:bool? product:?` | `getfilelist` |
| `/source/$project/$package/_history rev? meta:bool? deleted:bool? limit:num? startbefore:num?` | `getpackagehistory` |
| `/source/$project/$package/_meta rev? expand:bool? meta:bool? deleted:bool? view:? interconnect:?` | `getpackage` |
| `PUT:/source/$project/$package/_meta user:? comment:? requestid:num?` | `putpackage` |
| `DELETE:/source/$project/$package user:? comment:? requestid:num?` | `delpackage` |
| `/source/$project/$package/$filename rev? expand:bool? meta:bool? deleted:bool? view:? noajax:bool?` | `getfile` |
| `PUT:/source/$project/$package/$filename rev? user:? comment:? keeplink:bool? force:bool? meta:bool?` | `putfile` |
| `DELETE:/source/$project/$package/$filename rev? user:? comment:? keeplink:bool? force:bool? meta:bool?` | `delfile` |
| `/published view=collection name:* project*` | `published_collection` |
| `/published` | `published` |
| `/published/$project` | `published` |
| `/published/$project/$repository view=publishedpath medium:?` | `published_path` |
| `/published/$project/$repository view=status` | `published_status` |
| `/published/$project/$repository` | `published` |
| `/published/$project/$repository/$arch:filename view:?` | `published` |
| `/published/$project/$repository/$arch:filename/$filename view:?` | `published` |
| `/published/$project/$repository/$arch:filename/$filename/$subfilename:filename view:?` | `published` |
| `/getprojpack $project* $repository* $package* $arch? withrepos:bool? withsrcmd5:bool? withdeps:bool? withconfig:bool? expandedrepos:bool? ignoredisable:bool? nopackages:bool? withremotemap:bool? remotemaponly:bool? noremote:bool? parseremote:bool? buildinfo:bool? partition:? view:?` | `getprojpack` |
| `POST:/relsync $project $repository $arch` | `postrelsync` |
| `/relsync $project $repository $arch` | `getrelsync` |
| `/worker/$workerid` | `getworkercap` |
| `POST:/worker cmd=checkconstraints $project $repository $arch $package rev?` | `checkconstraints` |
| `/getsources $project $package $srcmd5:md5 workerid?` | `getsources` |
| `/getconfig $project $repository path:prp* workerid?` | `worker_getbuildconfig` |
| `/getsslcert $project autoextend:bool? workerid? signtype:? signflavor:?` | `getsslcert` |
| `/getbinaries $project $repository $arch binaries: nometa:bool? workerid? now:num? module*` | `worker_getbinaries` |
| `/getbinaryversions $project $repository $arch binaries: nometa:bool? workerid? now:num? module* withevr:bool?` | `worker_getbinaryversions` |
| `/getobsgendiffdata $project $repository $arch jobid? workerid?` | `getobsgendiffdata` |
| `/getmodulemd $project $package $srcmd5:md5 module* modularityplatform: modularitylabel: arch? view:?` | `getmodulemd` |
| `/getsignkey $project withpubkey:bool? autoextend:bool? withalgo:bool? signtype:? signflavor:?` | `getsignkey` |
| `/getpubkey $project autoextend:bool? signtype:? signflavor:?` | `getsignpubkey` |
| `!- /lastevents $filter:* start:num? obsname:? client:? block:bool?` | `lastevents` |
| `!- /lasteventsproxy $filter:* start:num? remoteurl: client:?` | `lasteventsproxy` |
| `/lastnotifications start:num? view:? block:bool? noprune:bool?` | `lastnotifications` |
| `/notificationpayload/$payloadkey:filename` | `getnotificationpayload` |
| `DELETE:/notificationpayload/$payloadkey:filename` | `deletenotificationpayload` |
| `POST:/event type: project: package:? repository:? arch:? job:?` | `newevent` |
| `/public/lastevents $filter:* start:num? obsname:? block:bool? client:?` | `lastevents` |
| `/search $in: $match: return:? values:bool?` | `search` |
| `/search/project $match:` | `search_proj` |
| `/search/project/id $match:` | `search_proj_id` |
| `/search/package $match:` | `search_pack` |
| `/search/package/id $match:` | `search_pack_id` |
| `POST:/search/published cmd:` | `search_published_updatedb` |
| `/search/published/binary/id $match: limit:num? withdownloadurl:bool? withscmsyncurl:bool?` | `search_published_binary_id` |
| `/search/published/binary/name $match: limit:num? withdownloadurl:bool? withscmsyncurl:bool?` | `search_published_binary_name` |
| `/search/published/pattern/id $match: limit:num? withdownloadurl:bool? withscmsyncurl:bool?` | `search_published_pattern_id` |
| `/search/published/repoinfo/id $match: limit:num? withdownloadurl:bool? withscmsyncurl:bool?` | `search_published_repoinfo_id` |
| `/service` | `listservices` |
| `PUT:/configuration` | `putconfiguration` |
| `/configuration` | `getconfiguration` |
| `PUT:/issue_trackers` | `putissuetrackers` |
| `POST:/issue_trackers cmd=issues` | `matchissues` |
| `/issue_trackers` | `getissuetrackers` |
| `/build` | `getprojectlist` |
| `/build/_workerstatus scheduleronly:bool? daemonsonly:bool? arch*` | `getworkerstatus` |
| `PUT:/build/_dispatchprios` | `putdispatchprios` |
| `/build/_dispatchprios` | `getdispatchprios` |
| `/build/_result scmrepository:? scmbranch:? code:? lastbuild:bool? multibuild:bool? locallink:bool? usetrackingbranch:bool?` | `getscmresult` |
| `/build/$project/_availablebinaries url:* path:prp* arch*` | `getavailablebinaries` |
| `POST:/build/$project cmd: repository* arch* package* code:* lastbuild:bool? wipe:* comment:? sysrq:?` | `docommand` |
| `/build/$project` | `getrepositorylist` |
| `/build/$project/_result oldstate:md5? view:resultview* lastbuild:bool? repository* arch* package* code:* multibuild:bool? locallink:bool?` | `getresult` |
| `/build/$project/_jobhistory package* code:* limit:num? endtime_start:num? endtime_end:num?` | `getjobhistory_project` |
| `/build/$project/$repository` | `getarchlist` |
| `/build/$project/$repository/_buildconfig path:prp*` | `getbuildconfig` |
| `/build/$project/$repository/$arch package* view:?` | `getpackagelist_build` |
| `!- /build/$project/$repository/$arch/_builddepinfo package* view:?` | `getbuilddepinfo` |
| `/build/$project/$repository/$arch/_jobhistory package* code:* limit:num? endtime_start:num? endtime_end:num?` | `getjobhistory` |
| `POST:/build/$project/$repository/$arch/_repository cmd=missingdodresources resource:* partition:?` | `missingdodresources` |
| `POST:/build/$project/$repository/$arch/_repository match:` | `postrepo` |
| `POST:/build/$project/$repository/$arch/$package cmd=copy oproject:project? opackage:package? orepository:repository? setupdateinfoid:? resign:bool? setrelease:? multibuild:bool?` | `copybuild` |
| `POST:/build/$project/$repository/$arch/$package` | `uploadbuild` |
| `/build/$project/$repository/$arch/$package_repository view:? binary:filename* nometa:bool? nosource:bool? withmd5:bool? module* withccache:bool? withevr:bool?` | `getbinarylist` |
| `POST:/build/$project/$repository/$arch/$package_repositorybuild/_buildinfo add:* debug:bool? striphdrmd5:bool?` | `getbuildinfo_post` |
| `/build/$project/$repository/$arch/$package/_buildinfo add:* internal:bool? debug:bool? striphdrmd5:bool?` | `getbuildinfo` |
| `/build/$project/$repository/$arch/$package/_jobstatus` | `getjobstatus` |
| `/build/$project/$repository/$arch/$package/_log nostream:bool? last:bool? lastsucceeded:bool? start:intnum? end:num? view:?` | `getlogfile` |
| `/build/$project/$repository/$arch/$package/_reason` | `getbuildreason` |
| `/build/$project/$repository/$arch/$package/_status` | `getbuildstatus` |
| `/build/$project/$repository/$arch/$package/_history limit:num?` | `getbuildhistory` |
| `/build/$project/$repository/$arch/$package/_buildstats limit:num?` | `getbuildstats` |
| `/build/$project/$repository/$arch/$package_repository/$filename view:? withfilelist:bool? module*` | `getbinary` |
| `PUT:/build/$project/$repository/$arch/_repository/$filename ignoreolder:bool? wipe:bool?` | `putbinary` |
| `DELETE:/build/$project/$repository/$arch/_repository/$filename` | `delbinary` |
| `/request/_lastid` | `getlastidrequest,	# just required for migration into api` |
| `/request/$id:num` | `getrequest, 		# just required for migration into api` |
| `/jobs/$arch partition:*` | `listjobs` |
| `POST:/notify/$_type: *:?` | `external_notification` |
| `POST:/notify_plugins/$_type:` | `notify_plugins` |
| `POST:/ownregistryrepo $project $repository $regrepo` | `registry_ownrepo` |
| `POST:/disownregistryrepo $project $repository $regrepo` | `registry_disownrepo` |
| `GET\|HEAD:/registry/$gunprefix:/$...:regrepo/_trust/tuf/$filename` | `notary_forward` |
| `GET\|HEAD:/registry/$...:regrepo/manifests/$manifest: ns:?` | `registry_forward` |
| `GET\|HEAD:/registry/$...:regrepo/manifestinfos/$manifest: ns:?` | `registry_manifestinfo` |
| `GET\|HEAD:/registry/$...:regrepo/blobs/$blob: ns:?` | `registry_forward` |
| `GET\|HEAD:/registry/$...:regrepo/signatures/$sig: ns:?` | `registry_forward` |
| `GET\|HEAD:/registry/$...:regrepo/referrers/$subject: ns:? artifactType:?` | `registry_forward` |
| `GET\|HEAD:/registry/$...:regrepo/tags/list n:intnum? last:? ns:?` | `registry_forward` |
| `GET\|HEAD:/registry/$...:regrepo/info.json ns:?` | `registry_forward` |
| `GET\|HEAD:/registry/_catalog n:intnum? last:? ns:?` | `registry_catalog` |
| `GET\|HEAD:/registry/info.json ns:?` | `registry_rootinfo` |
| `GET\|HEAD:/registry ns:?` | `registry_version` |
| `GET\|HEAD:/sigstore/$...:regrepo/$sig:` | `sigstore_forward` |
| `POST:/slsa cmd=addrefs $prpa` | `slsa_addrefs` |
| `/slsa/$project/$repository/$arch/$filename/$digest: view:?` | `slsa_getfile` |
| `/ajaxstatus aidx=all` | `getajaxstatuslist` |
| `/ajaxstatus aidx:num?` | `getajaxstatus` |
| `/serverstatus` | `BSStdServer::serverstatus` |
| `/` | `hello` |
| `/ajaxstatus` | `getajaxstatus` |
| `/build/$project/_result oldstate:md5? view:resultview* repository* arch* package* code:*` | `getresult` |
| `/build/$project/$repository/$arch package* view:?` | `getpackagelist_build` |
| `/build/$project/$repository/$arch/$package/_log nostream:bool? last:bool? lastsucceeded:bool? start:intnum? end:num?` | `getlogfile` |
| `/build/$project/$repository/$arch/$package_repository view:? binary:filename* nometa:bool? nosource:bool? withmd5:bool? module* withccache:bool? withevr:bool?` | `getbinarylist` |
| `/getbinaries $project $repository $arch binaries: nometa:bool? raw:bool? now:num? module*` | `worker_getbinaries` |
| `/getbinaryversions $project $repository $arch binaries: nometa:bool? now:num? module* withevr:bool?` | `worker_getbinaryversions` |
| `/getconfig $project $repository path:prp* workerid?` | `getbuildconfig_ajax` |
| `/lastevents $filter:* start:num? obsname:? client:?` | `lastevents` |
| `/lasteventsproxy $filter:* start:num? remoteurl: client:?` | `lasteventsproxy` |
| `/lastnotifications start:num? view:? block:bool?` | `lastnotifications` |
| `/source/$project/$package cmd=waitservice servicemark:` | `waitservicerun` |
| `/source/$project/$package rev view:` | `getfilelist_ajax` |
| `/source/$project/$package:package/$filename rev?` | `getfile` |
| `/request/$id:num withkey:bool? oldkey:md5?` | `getrequest` |
| `/sourcediffcache/$cacheid:md5 view:? file:*` | `getsourcediffcache` |

## bs_repserver (88 routes)

| Route | Perl handler |
|---|---|
| `/` | `hello` |
| `!rw :` | `undef` |
| `!- GET:` | `undef` |
| `!- HEAD:` | `undef` |
| `POST:/build/$project cmd=move oproject:project` | `moveproject` |
| `POST:/build/$project/$repository/$arch/_repository cmd=missingdodresources resource:* partition:?` | `missingdodresources` |
| `POST:/build/$project/$repository/$arch/_repository match:` | `postrepo` |
| `/build/$project/$repository/$arch package* view:? withbininfocookie:bool?` | `getpackagelist_build` |
| `/build/$project/$repository/$arch/_builddepinfo package* view:?` | `getbuilddepinfo` |
| `/build/$project/$repository/$arch/_jobhistory package* code:* limit:num? endtime_start:num? endtime_end:num?` | `getjobhistory` |
| `POST:/build/$project/$repository/$arch/_relsync` | `postrelsync` |
| `/build/$project/$repository/$arch/_relsync` | `getrelsync` |
| `POST:/build/$project/$repository/$arch/$package cmd=copy oproject:project? opackage:package? orepository:repository? setupdateinfoid:? resign:bool? setrelease:? remoteurl:?` | `copybuild` |
| `POST:/build/$project/$repository/$arch/$package` | `uploadbuild` |
| `!worker,rw /build/$project/$repository/$arch/$package:package_repository view:? binary:filename* nometa:bool? noajax:bool? nosource:bool? noimport:bool? withmd5:bool? module* withccache:bool? withannotation:bool? withevr:bool? copybuild:bool? aggregatemode:bool?` | `getbinarylist` |
| `POST:/build/$project/$repository/$arch/$package_repositorybuild/_buildinfo add:* internal:bool? debug:bool? deps:bool?` | `getbuildinfo_post` |
| `/build/$project/$repository/$arch/$package/_buildinfo add:* internal:bool? debug:bool? deps:bool?` | `getbuildinfo` |
| `/build/$project/$repository/$arch/$package/_reason` | `getbuildreason` |
| `/build/$project/$repository/$arch/$package/_status` | `getbuildstatus` |
| `/build/$project/$repository/$arch/$package/_jobstatus` | `getjobstatus` |
| `/build/$project/$repository/$arch/$package/_history limit:num?` | `getbuildhistory` |
| `/build/$project/$repository/$arch/$package/_buildstats limit:num?` | `getbuildstats` |
| `/build/$project/$repository/$arch/$package/_log nostream:bool? start:intnum? end:num? handoff:bool? last:bool? lastsucceeded:bool? view:?` | `getlogfile` |
| `/build/$project/$repository/$arch/$package:package_repository/$filename view:? withfilelist:bool? module*` | `getbinary` |
| `PUT:/build/$project/$repository/$arch/_repository/$filename ignoreolder:bool? wipe:bool?` | `putbinary` |
| `DELETE:/build/$project/$repository/$arch/_repository/$filename` | `delbinary` |
| `PUT:/build/_dispatchprios` | `putdispatchprios` |
| `/build/_dispatchprios` | `getdispatchprios` |
| `POST:/event $type: $project $package? repository? arch? job? worker:job?` | `forwardevent` |
| `/worker/$workerid` | `getworkercap` |
| `POST:/worker cmd=checkconstraints $project $repository $arch $package` | `checkconstraints` |
| `!worker /worker $arch $port $state: workerid? working:bool? memory:num? disk:num? buildarch:arch* tellnojob:bool? proto:?` | `workerstate` |
| `!worker /getbuildcode` | `getbuildcode` |
| `!worker /getworkercode` | `getworkercode` |
| `!worker POST:/putjob $arch $job $jobid $code:? now:num? kiwitree:bool? workerid?` | `putjob` |
| `!worker POST:/workerdispatched $arch $job $jobid hostarch:arch port proto:? workerid?` | `workerdispatched` |
| `!worker /getbinaries $project $repository $arch binaries: nometa:bool? metaonly:bool? workerid? now:num? module* withannotation:bool?` | `getbinaries` |
| `!worker /getbinaryversions $project $repository $arch binaries: nometa:bool? workerid? now:num? module* withevr:bool?` | `getbinaryversions` |
| `!worker /getjobdata $arch $job $jobid workerid?` | `getjobdata` |
| `!worker /getpackagebinaryversionlist $project $repository $arch $package* withcode:bool? workerid?` | `getpackagebinaryversionlist` |
| `!worker /badpackagebinaryversionlist $project $repository $arch $package* workerid?` | `badpackagebinaryversionlist` |
| `!worker /getpreinstallimageinfos $prpa+ match:? workerid?` | `getpreinstallimageinfos` |
| `/getobsgendiffdata $project $repository $arch jobid? workerid?` | `getobsgendiffdata` |
| `/published` | `publisheddir` |
| `/published/$project` | `publisheddir` |
| `/published/$project/$repository view=status` | `published_status` |
| `/published/$project/$repository` | `publisheddir` |
| `/published/$project/$repository/$arch:filename view:?` | `publisheddir` |
| `/published/$project/$repository/$arch:filename/$filename view:?` | `publishedfile` |
| `/published/$project/$repository/$arch:filename/$filename/$subfilename:filename view:?` | `publishedfile` |
| `/jobs` | `listjobarchs` |
| `POST:/jobs/_mdload` | `postmdload` |
| `/jobs/$arch` | `listjobs` |
| `PUT:/jobs/$arch/$job` | `addjob` |
| `POST:/jobs/$arch/$job cmd=idleworker workerid jobid?` | `idleworkerjob` |
| `POST:/jobs/$arch/$job cmd=setdispatchdetails details:?` | `setdispatchdetails` |
| `POST:/jobs/$arch/$job cmd=fail message:` | `failjob` |
| `DELETE:/jobs/$arch/$job` | `deljob` |
| `/jobs/$arch/$job view:?` | `getjob` |
| `/workerstatus daemonsonly:bool? arch* type:*` | `workerstatus` |
| `GET\|HEAD:/registry/$gunprefix:/$...:regrepo/_trust/tuf/$filename` | `registry_tuf` |
| `GET\|HEAD:/registry/$...:regrepo/manifests/$manifest:` | `registry_manifest` |
| `GET\|HEAD:/registry/$...:regrepo/blobs/$blob:` | `registry_blob` |
| `GET\|HEAD:/registry/$...:regrepo/signatures/$sig:` | `registry_sigstore` |
| `GET\|HEAD:/registry/$...:regrepo/referrers/$subject: artifactType:?` | `registry_referrers` |
| `GET\|HEAD:/registry/$...:regrepo/manifestinfos/$manifest:` | `registry_manifestinfo` |
| `GET\|HEAD:/registry/$...:regrepo/tags/list n:intnum? last:?` | `registry_taglist` |
| `GET\|HEAD:/registry/$...:regrepo/info.json` | `registry_info` |
| `GET\|HEAD:/registry/_catalog` | `registry_catalog` |
| `GET\|HEAD:/registry` | `registry_version` |
| `POST:/slsa cmd=addrefs $prpa` | `slsa_addrefs` |
| `/slsa/$project/$repository/$arch/$filename/$digest: view:?` | `slsa_getfile` |
| `PUT:/configuration` | `putconfiguration` |
| `/configuration` | `getconfiguration` |
| `/_result $prpa+ oldstate:md5? package* code:* lastbuild:bool? withbinarylist:bool? withstats:bool? withinfo:bool? withreproduciblecheck:bool? summary:bool? withversrel:bool?` | `getresult` |
| `/_jobhistory $prpa+ package* code:* limit:num? endtime_start:num? endtime_end:num?` | `getjobhistory_project` |
| `POST:/_command $cmd: $prpa+ package* code:* lastbuild:bool? sysrq:? allarch:bool?` | `docommand` |
| `/serverstatus` | `BSStdServer::serverstatus` |
| `/ajaxstatus aidx=all` | `getajaxstatuslist` |
| `/ajaxstatus aidx:num?` | `getajaxstatus` |
| `/` | `hello` |
| `/ajaxstatus` | `getajaxstatus` |
| `/build/$project/$repository/$arch/$package/_log nostream:bool? last:bool? lastsucceeded:bool? start:intnum? end:num? view:?` | `getlogfile` |
| `/build/$project/$repository/$arch/$package:package_repository view:? binary:filename* nometa:bool? nosource:bool? module* withccache:bool? withannotation:bool?` | `getbinarylist` |
| `/build/$project/$repository/$arch/$package:package_repository/$filename view:? withfilelist:bool? module*` | `getbinary` |
| `/_result $prpa+ oldstate:md5? package* code:* withbinarylist:bool? withstats:bool? withinfo:bool? withreproduciblecheck:bool? summary:bool? withversrel:bool?` | `getresult` |
| `/getbinaries $project $repository $arch binaries: nometa:bool? metaonly:bool? now:num? module* withannotation:bool?` | `getbinaries` |
| `/getbinaryversions $project $repository $arch binaries: nometa:bool? now:num? module* withevr:bool?` | `getbinaryversions` |

## bs_service (8 routes)

| Route | Perl handler |
|---|---|
| `/` | `hello` |
| `!rw :` | `undef` |
| `!- GET:` | `undef` |
| `!- HEAD:` | `undef` |
| `/service` | `list_service` |
| `/serverstatus` | `BSStdServer::serverstatus` |
| `!- POST:/sourceupdate/$project/$package $timeout:num? noprefix:bool?` | `run_source_update` |
| `PUT:/configuration` | `putconfiguration` |

## bs_getbinariesproxy (4 routes)

| Route | Perl handler |
|---|---|
| `/` | `hello` |
| `/getbinaries $project $repository $arch binaries: nometa:bool? metaonly:bool? module* workerid? now:num? server:` | `getbinaries` |
| `/getbinaries_kiwiproduct $project $repository $arch $package binary:filename+ noajax:bool? workerid? now:num? server:` | `getbinaries_kiwiproduct` |
| `/getpreinstallimage $prpa $hdrmd5:md5 path: sizek:num? workerid? now:num? server:` | `getpreinstallimage` |

