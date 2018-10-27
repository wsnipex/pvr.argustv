import hudson.plugins.throttleconcurrents.ThrottleJobProperty;
import java.io.File
import java.util.regex.Matcher

properties([
	parameters([
		string(defaultValue: '1', description: 'debian package revision tag', name: 'TAGREV', trim: true),
		choice(choices: ['all', 'cosmic', 'bionic', 'xenial'], description: 'Ubuntu version to build for', name: 'dists'),
		choice(choices: ['auto', 'wsnipex-test', 'nightly', 'unstable', 'stable'], description: 'PPA to use', name: 'PPA')
	])
])

/**
 * Simple wrapper step for building a plugin
 */
def buildPlugin(Map addonParams = [:])
{
	def PLATFORMS_VALID = [
		'android-armv7': 'android',
		'android-aarch64': 'android-arm64-v8a',
		'ios-armv7': 'ios',
		'ios-aarch64': 'ios',
		'osx-x86_64': 'osx64',
		'ubuntu-ppa': 'linux',
		'windows-i686': 'windows/win32',
		'windows-x86_64': 'windows/x64'
	]
	def VERSIONS_VALID = [
		'master': 'leia',
		'Leia': 'leia'
	]
	def PPAS_VALID = [
		'nightly': 'ppa:team-xbmc/xbmc-nightly',
		'unstable': 'ppa:team-xbmc/unstable',
		'stable': 'ppa:team-xbmc/ppa',
		'wsnipex-test': 'ppa:wsnipex/xbmc-addons-test'
	]
	def PPA_VERSION_MAP = [
		'master': 'nightly',
		'Leia': 'unstable'
	]

	def platforms = addonParams.containsKey('platforms') && addonParams.platforms.metaClass.respondsTo('each') && addonParams.platforms.every{ p -> p in PLATFORMS_VALID } ? addonParams.platforms : PLATFORMS_VALID.keySet()
	def version = addonParams.containsKey('version') && addonParams.version in VERSIONS_VALID ? addonParams.version : VERSIONS_VALID.keySet()[0]
	def addon = env.JOB_NAME.tokenize('/')[1]
	def dists = params.dists == "all" ? ["bionic", "xenial", "cosmic"] : [params.dists]
	def ppa = params.PPA == "auto" ? PPAS_VALID[PPA_VERSION_MAP[version]] : PPAS_VALID[params.PPA]
	def packageversion = ""
	Map tasks = [failFast: false]

	env.Configuration = 'Release'


	for (int i = 0; i < platforms.size(); ++i)
	{
		String platform = platforms[i]
		if (platform == 'ubuntu-ppa') continue

		def category = "binary-addons/${platform}-${version}"
		if (ThrottleJobProperty.fetchDescriptor().getCategories().every{ c -> c.getCategoryName() !=  category})
		{
			ThrottleJobProperty.fetchDescriptor().getCategories().add(new ThrottleJobProperty.ThrottleCategory(category, 1, 0, null));
			ThrottleJobProperty.fetchDescriptor().save()
		}
		tasks[platform] = {
			throttle(["binary-addons/${platform}-${version}"])
			{
				node(platform)
				{
					ws("workspace/binary-addons/kodi-${platform}-${version}")
					{
						stage("prepare (${platform})")
						{
							pwd = pwd()
							checkout([
								changelog: false,
								$class: 'GitSCM',
								branches: [[name: "* /${version}"]],
								doGenerateSubmoduleConfigurations: false,
								extensions: [[$class: 'CloneOption', 'honorRefspec': true, 'noTags': true, 'reference': "${pwd}/../../kodi"]],
								userRemoteConfigs: [[refspec: "+refs/heads/${version}:refs/remotes/origin/${version}", url: 'https://github.com/xbmc/xbmc.git']]
							])

							if (isUnix())
							{
								folder = PLATFORMS_VALID[platform]
								sh "WORKSPACE=`pwd` sh -xe ./tools/buildsteps/${folder}/prepare-depends"
								folder = PLATFORMS_VALID[platform]
								sh "WORKSPACE=`pwd`" + (platform == 'ios-aarch64' ? ' DARWIN_ARM_CPU=arm64' : '') + " sh -xe ./tools/buildsteps/${folder}/configure-depends"
								folder = PLATFORMS_VALID[platform]
								sh "WORKSPACE=`pwd` sh -xe ./tools/buildsteps/${folder}/make-native-depends"
								sh "git clean -xffd -- tools/depends/target/binary-addons"
							}
							else
							{
								folder = PLATFORMS_VALID[platform]
								bat "tools/buildsteps/${folder}/prepare-env.bat"
								folder = PLATFORMS_VALID[platform]
								bat "tools/buildsteps/${folder}/download-dependencies.bat"
								bat "git clean -xffd -- tools/depends/target/binary-addons"
							}

							dir("tools/depends/target/binary-addons/${addon}")
							{
								if (env.BRANCH_NAME)
								{
									def scmVars = checkout(scm)
									currentBuild.displayName = scmVars.GIT_BRANCH + '-' + scmVars.GIT_COMMIT.substring(0, 7)
								}
								else if ((env.BRANCH_NAME == null) && (repo))
								{
									git repo
								}
								else
								{
									error 'buildPlugin must be used as part of a Multibranch Pipeline *or* a `repo` argument must be provided'
								}
							}

							dir("tools/depends/target/binary-addons/addons/${addon}")
							{
								writeFile file: "${addon}.txt", text: "${addon} . ."
								writeFile file: 'platforms.txt', text: 'all'
							}
						}

						stage("build (${platform})")
						{
							dir("tools/depends/target/binary-addons")
							{
								if (isUnix())
									sh "make -j $BUILDTHREADS ADDONS='${addon}' ADDONS_DEFINITION_DIR=`pwd`/addons ADDON_SRC_PREFIX=`pwd` EXTRA_CMAKE_ARGS=\"-DPACKAGE_DIR=`pwd`/../../../../cmake/addons/build/zips\" PACKAGE=1"
							}

							if (!isUnix())
							{
								env.ADDONS_DEFINITION_DIR = pwd().replace('\\', '/') + '/tools/depends/target/binary-addons/addons'
								env.ADDON_SRC_PREFIX = pwd().replace('\\', '/') + '/tools/depends/target/binary-addons'
								folder = PLATFORMS_VALID[platform]
								bat "tools/buildsteps/${folder}/make-addons.bat package ${addon}"
							}

							if (isUnix())
								sh "grep '${addon}' cmake/addons/.success"
						}

						stage("archive (${platform})")
						{
							archiveArtifacts artifacts: "cmake/addons/build/zips/${addon}+${platform}/${addon}-*.zip"
						}

						stage("deploy (${platform})")
						{
							if (env.TAG_NAME != null)
							{
								echo "Deploying: ${addon} ${env.TAG_NAME}"
								versionFolder = VERSIONS_VALID[version]
								sshPublisher(
									publishers: [
										sshPublisherDesc(
											configName: 'Mirrors',
											transfers: [
												sshTransfer(
													execCommand: """\
mkdir -p /home/git/addons-binary/${versionFolder}
chmod 444 upload/${addon}+${platform}/${addon}-*.zip
(mv upload/${addon}+${platform}/ /home/git/addons-binary/${versionFolder}/ || cp upload/${addon}+${platform}/${addon}-*.zip /home/git/addons-binary/${versionFolder}/${addon}+${platform}/) 2> /dev/null
PUBLISHED=\$?
if [ \$PUBLISHED -ne 0 ]; then
	echo `ls upload/${addon}+${platform}/${addon}-*.zip | cut -d / -f 2-` was already published >&2
fi
rm -fr upload/${addon}+${platform}/ 2> /dev/null
exit \$PUBLISHED
""",
													remoteDirectory: 'upload',
													removePrefix: 'cmake/addons/build/zips/',
													sourceFiles: "cmake/addons/build/zips/${addon}+${platform}/${addon}-*.zip"
												)
											]
										)
									]
								)
							}
						}
					}
				}
			}
		}
	}

	tasks["ubuntu-ppa"] = {
		throttle(["binary-addons/ubuntu-ppa-${version}"])
		{
			node("linux")
			{
				ws("workspace/binary-addons/kodi-ubuntu-ppa-${version}")
				{
					stage("clone")
					{
						dir("${addon}")
						{
							if (env.BRANCH_NAME)
							{
								def scmVars = checkout(scm)
								currentBuild.displayName = scmVars.GIT_BRANCH + '-' + scmVars.GIT_COMMIT.substring(0, 7)
							}
							else if ((env.BRANCH_NAME == null) && (repo))
							{
								git repo
							}
							else
							{
								error 'buildPlugin must be used as part of a Multibranch Pipeline *or* a `repo` argument must be provided'
							}
						}
					}
					
					stage("build")
					{
						dir("${addon}")
						{
							echo "Ubuntu dists enabled: ${dists} - TAGREV: ${params.TAGREV} - PPA: ${params.PPA}"
							def addonsxml = readFile "${addon}/addon.xml.in"
							packageversion = getVersion(addonsxml)
							echo "Detected PackageVersion: ${packageversion}"
							def changelogin = readFile 'debian/changelog.in'
							def origtarball = 'kodi-' + addon.replace('.', '-') + "_${packageversion}.orig.tar.gz"
							sh "git archive --format=tar.gz -o ../${origtarball} HEAD"

							for (dist in dists)
							{
								echo "building for ${dist}"
								def changelog = changelogin.replace('#PACKAGEVERSION#', packageversion).replace('#TAGREV#', params.TAGREV).replace('#DIST#', dist)
								writeFile file: "debian/changelog", text: "${changelog}"
								sh "debuild -S -k'jenkins (jenkins build bot) <jenkins@kodi.tv>'"
							}
						}
					}
					stage("upload")
					{
						def changespattern = 'kodi-' + addon.replace('.', '-') + "_${packageversion}-${params.TAGREV}*_source.changes"
						sh "dput ${ppa} ${changespattern}"
					}
				}
  			}
		}
	}

	parallel(tasks)
}

@NonCPS
def getVersion(text) {
	def matcher = text =~ /version=\"([\d.]+)\"/
        matcher ? matcher.getAt(1)[1] : null
}

buildPlugin()
