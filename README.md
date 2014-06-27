libgdx-one-click-deployment
===========================

First we include this duplicate code in the _android/build.gradle_ and _ios/build.gradle_:

 * A __versioning software reader__:
 
```
// Read current version from properties file
ext.versionFile = file('version.properties')

task loadVersion {
    project.version = readVersion()
}

ProjectVersion readVersion() {
    logger.quiet 'Reading the android version file.'

    if (!versionFile.exists()) {
        throw new GradleException("Required version file does not exit: $versionFile.canonicalPath")
    }

    Properties versionProps = new Properties()

    versionFile.withInputStream { stream ->
        versionProps.load(stream)
    }

    new ProjectVersion(versionProps.major.toInteger(), versionProps.minor.toInteger(), versionProps.patch.toInteger(), versionProps.build.toInteger())
}
```

 * __Auto-increment version tasks__:
 
```
// Control version
task incrementMajorVersion(group: 'versioning', description: 'Increments project major version.') << {
    String currentVersion = version.toString()
    ++version.major
	// Reset minor and patch
	version.minor = 0
	version.patch = 0
    String newVersion = version.toString()
    logger.info "Incrementing major project version: $currentVersion -> $newVersion"

    ant.propertyfile(file: versionFile) {
        entry(key: 'major', type: 'int', operation: '+', value: 1)
		entry(key: 'minor', type: 'int', operation: '=', value: 0)
		entry(key: 'patch', type: 'int', operation: '=', value: 0)
    }
}

task incrementMinorVersion(group: 'versioning', description: 'Increments project minor version.') << {
    String currentVersion = version.toString()
    ++version.minor
	// Reset patch
	version.patch = 0
    String newVersion = version.toString()
    logger.info "Incrementing minor project version: $currentVersion -> $newVersion"

    ant.propertyfile(file: versionFile) {
        entry(key: 'minor', type: 'int', operation: '+', value: 1)
		entry(key: 'patch', type: 'int', operation: '=', value: 0)
    }
}

task incrementPatchVersion(group: 'versioning', description: 'Increments project patch version.') << {
    String currentVersion = version.toString()
    ++version.patch
    String newVersion = version.toString()
    logger.info "Incrementing patch project version: $currentVersion -> $newVersion"

    ant.propertyfile(file: versionFile) {
        entry(key: 'patch', type: 'int', operation: '+', value: 1)
    }
}

task incrementBuildVersion(group: 'versioning', description: 'Increments project build version.') << {
    String currentVersion = version.toString()
    ++version.build
    String newVersion = version.toString()
    logger.info "Incrementing build project version: $currentVersion -> $newVersion"

    ant.propertyfile(file: versionFile) {
        entry(key: 'build', type: 'int', operation: '+', value: 1)
    }
}
```

 * At the end a __ProjectVersion Class Definition__:
 
```
// Class definition
class ProjectVersion {
	
    Integer major
	
    Integer minor
	
    Integer patch
	
    Integer build

    ProjectVersion(Integer major, Integer minor, Integer patch, Integer build) {
        this.major = major
        this.minor = minor
        this.patch = patch
        this.build = build
    }

    String getVersionName() {
        this.major + "." + this.minor + "." + this.patch
    }
	
    String getVersionCode() {
		Integer.parseInt(new Date().format('yyyyMMdd')) + this.build
    }

    @Override
    String toString() {
        this.getVersionName() + "_" + this.getVersionCode()
    }
}

```

Then both in the _android_ and _ios_ we create two __properties files__ called `version.properties` that contains this:

```
major=0
minor=0
patch=0
build=0
```

And a __global properties__ at the root of our project file look like this:

```
# Testflight configuration
iosBuildPath=ios/build/robovm/
iosAppName=IOSLauncher
testflightURL=http://testflightapp.com/api/builds.json
testflightAT=
testflightTT=
testflightDefaultNote=This build was uploaded via the upload API and Gradle
testflightNotify=True
testflightDL=<Distribute list names separates by commas>
# Testfairy configuration
androidBuildPath=android/build/apk/
testfairyURL=https://app.testfairy.com/api/upload
testfairyAK=
androidAppName=android-release
testfairyProguardPath=android/proguard-project.txt
testfairyTG=<Distribute groups separates by commas>
testfairyMetrics=cpu,memory,network,logcat
testfairyVideo=off
testfairyMD=10m
testfairyVQ=low
testfairyVR=1.0
testfairyIW=off
testfairyComment=This build was uploaded via the upload API and Gradle
```

Next we configure individual tasks for _android/build.gradle_.

Locate project `android` and add a __signing configuration__ to sign our game with default android keystore values. It should look like this:

```
android {
    
	// ...
	
    signingConfigs {
        release {
            storeFile file( System.getProperty("user.home") + "/.android/debug.keystore")
            storePassword "android"
            keyAlias "androiddebugkey"
            keyPassword "android"
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
        }
    }

}
```

Next do you insert a dependency before `preReleaseBuild` to auto-increment build version with each build.

```
task updateAndroidManifestXML(group: 'versioning', description: 'Updates AndroidManifest.xml project file.') << {
	
    def manifestFile = file("AndroidManifest.xml")
    
	def pattern = java.util.regex.Pattern.compile("versionCode=\"(\\d+)\"")
    def manifestContent = manifestFile.getText()
    def matcher = pattern.matcher(manifestContent)
    matcher.find()
    manifestContent = matcher.replaceFirst("versionCode=\"" + version.getVersionCode() + "\"")
	
	pattern = java.util.regex.Pattern.compile("versionName=\"(.*)\"")
    matcher = pattern.matcher(manifestContent)
    matcher.find()
    manifestContent = matcher.replaceFirst("versionName=\"" + version.getVersionName() + "\"")
    
	manifestFile.write(manifestContent)
	
}

// Autoincrement build versioning
updateAndroidManifestXML.dependsOn incrementBuildVersion

tasks.whenTaskAdded { task ->
	if( task.name == 'preReleaseBuild' )
		task.dependsOn updateAndroidManifestXML
}
```

And finally we include a task to upload and distribute our game using __testfairy__

```
task execTestfairyUpload (type:Exec) {
	
	def currentWorkingDir = System.getProperty("user.dir")
	commandLine 'curl'
	args testfairyURL,
		 '-F',
		 'api_key=' + testfairyAK,
		 '-F',
		 'apk_file=@' + currentWorkingDir + '/' + androidBuildPath + androidAppName + '.apk',
		 '-F',
		 'proguard_file=@' + currentWorkingDir + '/' + testfairyProguardPath,
		 '-F',
		 'testers_groups=' + testfairyTG,
		 '-F',
		 'metrics=' + testfairyMetrics,
		 '-F',
		 'max-duration=' + testfairyMD,
		 '-F',
		 'video=' + testfairyVideo,
		 '-F',
		 'video-quality=' + testfairyVQ,
		 '-F',
		 'video-rate=' + testfairyVR,
		 '-F',
		 'icon-watermark=' + testfairyIW,
		 '-F',
		 'comment=' + testfairyComment
	//store the output instead of printing to the console:
	standardOutput = new ByteArrayOutputStream()

	//extension method execTestfairyUpload.output() can be used to obtain the output:
	ext.output = {
		return standardOutput.toString()
	}
	
}

task testfairyUpload(dependsOn: execTestfairyUpload) << {
	logger.info "${execTestfairyUpload.output()}"
}
```

It is now the turn of _ios/build.gradle_. Open this and __modify createIPA task__ so it depends on update of _robovm.properties_.
As we shall she below `updateRoboVMProperties` task will update `app.version` and `app.build`.
Also `updateRoboVMProperties` task depends on a increment build version task:

```
- createIPA.dependsOn build

+ // Update info.plist.xml
+ task updateRoboVMProperties << {
+ 	
+     ant.propertyfile(file: robovmFile) {
+         entry(key: 'app.version', type: 'string', operation: '=', value: version.getVersionName())
+     }
+ 	
+     ant.propertyfile(file: robovmFile) {
+         entry(key: 'app.build', type: 'int', operation: '=', value: version.build.toString())
+     }
+ 	
+ }

+ // Autoincrement build versioning
+ incrementBuildVersion.dependsOn build
+ updateRoboVMProperties.dependsOn incrementBuildVersion
+ createIPA.dependsOn updateRoboVMProperties
```

And also finally we include a task to __zip dSYM__, upload and distribute our game using __testflight__ in this occasion

```
task execTestflightZip (type:Exec) {
	
	def currentWorkingDir = System.getProperty("user.dir")
	commandLine '/usr/bin/zip'
	args '-r',
		 currentWorkingDir + '/' + iosBuildPath + 'IOSLauncher.app.dSYM.zip',
		 currentWorkingDir + '/' + iosBuildPath + 'IOSLauncher.app.dSYM'
	//store the output instead of printing to the console:
	standardOutput = new ByteArrayOutputStream()

	//extension method execTestflightDistribute.output() can be used to obtain the output:
	ext.output = {
		return standardOutput.toString()
	}
	
}

task testflightZip(dependsOn: execTestflightZip) << {
	logger.info "${execTestflightZip.output()}"
}

task execTestflightUpload (type:Exec, dependsOn: testflightZip) {
	
	def currentWorkingDir = System.getProperty("user.dir")
	commandLine 'curl'
	args testflightURL,
		 '-F',
		 'file=@' + currentWorkingDir + '/' + iosBuildPath + iosAppName + '.ipa',
		 '-F',
		 'dsym=@' + currentWorkingDir + '/' + iosBuildPath + iosAppName + '.app.dSYM.zip',
		 '-F',
		 'api_token=' + testflightAT,
		 '-F',
		 'team_token=' + testflightTT,
		 '-F',
		 'notes=' + testflightDefaultNote,
		 '-F',
		 'notify=' + testflightNotify,
		 '-F',
		 'distribution_lists=' + testflightDL
	//store the output instead of printing to the console:
	standardOutput = new ByteArrayOutputStream()

	//extension method execTestflightDistribute.output() can be used to obtain the output:
	ext.output = {
		return standardOutput.toString()
	}
	
}

task testflightUpload(dependsOn: execTestflightUpload) << {
	logger.info "${execTestflightUpload.output()}"
}
```

## Distribute anywhere ##

Lack of resources is a major problem that faces an studio indie.

Package and distribute tasks only need commandline so that we'll __configurate our MAC to access remotely by SSH__ from any system, including your [phone](https://play.google.com/store/apps/details?id=jackpal.androidterm&hl=es) for an emergency.

There are hundreds of tutorials on how to do this. Here's an [example](http://www.maclife.com/article/howtos/how_enable_ssh_your_mac). I'm just giving an example of the potential for our approach.

Make sure you have the [Mac environment](https://github.com/libgdx/libgdx/wiki/Gradle-on-the-Commandline) properly configured according to libGDX WIKI and here a set of examples:

```
ssh user@server
// First increment minor version, increment build version, create *.ipa and distribute using testflight
server:~ user$ ./gradlew ios:incrementMinorVersion ios:createIPA ios:testflightUpload
// Increment build version, create *.apk and distribute using testfairy
server:~ user$ ./gradlew android:assembleRelease android:testfairyUpload
```

__IMPORTANT: Testers with tethered phones can't work with testflight.__ An alternative is distribute our *.ipa with an a cloud storage service and they install using [iFunbox](http://www.igeeksblog.com/ifunbox-the-best-installous-alternative-to-install-ipa-files/).
