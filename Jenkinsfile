#!groovy

// only 40 builds
properties([buildDiscarder(logRotator(artifactNumToKeepStr: '40', numToKeepStr: '40'))])

node ('docker') {

    def DEFAULT_REPO = 'https://github.com/jenkinsci/blueocean-plugin.git'
    def DEFAULT_BUILD_NUM = 'latest'
    def NO_BUILD_NUM = ''

    // Allow the pipeline to be built with parameters, defaulting the
    // Blue Ocean branch name to be that of the ATH branch name. If no such branch
    // of Blue Ocean exists, then the ATH will just run against the master branch of
    // Blue Ocean.
    properties([parameters([
            string(name: 'BLUEOCEAN_REPO_URL', defaultValue: DEFAULT_REPO, description: 'The Blue Ocean repository against which the tests on this ATH branch will run. If you want to validate a fork, you can change this.'),
            string(name: 'BLUEOCEAN_BRANCH_NAME', defaultValue: "${env.BRANCH_NAME}", description: 'Blue Ocean branch name (on the above repository) against which the tests on this ATH branch will run.'),
            string(name: 'BUILD_NUM', defaultValue: DEFAULT_BUILD_NUM, description: 'The Blue Ocean build number from the CI server. Used to get pre-assembled Jenkins plugins Vs building (see above repo settings). Use a valid build number, or "latest" to get artifacts from the latest build. Otherwise, leave blank to build the latest code from the branch.<br/><strong>NOTE:</strong> Uses the above BLUEOCEAN_BRANCH_NAME to determine the upstream build Job name from which to get the pre-assembled archives.')
    ]), pipelineTriggers([])])

    def repoUrl;
    def branchName;
    def buildNumber;
    def LAST_COMPLETED_BUILD_SELSECTOR = [$class: 'LastCompletedBuildSelector'];

    try {
        repoUrl = "${BLUEOCEAN_REPO_URL}"
        branchName = "${BLUEOCEAN_BRANCH_NAME}"
        buildNumber = "${BUILD_NUM}"
    } catch (e) {
        echo "********************************************************************************************"
        echo "The build parameters for this branch are not yet initialized (or were modified)."
        echo "Will attempt to run the ATH against the pre-assembled HPIs from the latest build of."
        echo "${env.BRANCH_NAME}, falling back to the master branch build if there are no builds"
        echo "on that branch Job."
        echo "********************************************************************************************"
        repoUrl = DEFAULT_REPO
        branchName = env.BRANCH_NAME
        buildNumber = DEFAULT_BUILD_NUM
    }

    stage 'init'
    checkout scm

    // Run selenium in a docker container of its own on the host.
    sh "./start-selenium.sh"

    // Encode the branch name. Need to do it out here because of JENKINS-34973.
    def urlEncodedBranchName = URLEncoder.encode(branchName, "UTF-8").replace("+", "%20");

    try {
        // Build an image from the the local Dockerfile
        sh 'docker build -t blueocean-ath-builder --build-arg GID=$(id -g ${USER}) --build-arg UID=$(id -u ${USER}) .'

        def athImg = docker.image('blueocean-ath-builder')

        // fetch Maven configuration managed by Config File Provider plugin
        // to change it, go to Manage Jenkins > Managed files
        configFileProvider([configFile(fileId: 'blueocean-maven-settings', targetLocation: 'settings.xml')]) {

        //
        // Run the build container, giving it the same network stack as the selenium
        // container.
        //
        // To bind in the local ~/.m2 when running in dev mode, simply add the following
        // volume binding to the "inside" container run settings (change username from "tfennelly"):
        //       -v /home/tfennelly/.m2:/home/bouser/.m2
        //
        athImg.inside("--net=container:blueo-selenium") {
            try {
                sh "echo 'Starting build stage'"
                // Build blueocean and the ATH
                stage 'build'
                if (buildNumber == NO_BUILD_NUM) {
                    // This build of the ATH was not triggered from an upstream build of blueocean itself
                    // so we must get and build blueocean.
                    dir('blueocean-plugin') {
                        // Try checking out the Blue Ocean branch having the name supplied by build parameter. If that fails
                        // (i.e. doesn't exist ), just use the default/master branch and run the ATH tests against that.
                        try {
                            git(url: "${repoUrl}", branch: "${branchName}")
                            echo "Found a Blue Ocean branch named '${branchName}'. Running ATH against that branch."
                        } catch (Exception e) {
                            echo "No Blue Ocean branch named '${branchName}'. Running ATH against 'master' instead."
                            branchName = "master";
                            git(url: 'https://github.com/jenkinsci/blueocean-plugin.git', branch: "master")
                        }
                        // Need test-compile because the rest-impl has a test-jar that we
                        // need to make sure gets compiled and installed for other modules.
                        // Must cd into blueocean-plugin before running build
                        // see https://issues.jenkins-ci.org/browse/JENKINS-33510
                        sh "cd blueocean-plugin && mvn -B clean test-compile install -DskipTests -s ../settings.xml"
                    }
                } else {
                    def selector;

                    // Get the ATH plugin set from an upstream build of the "blueocean" job. All blueocean builds
                    // already have the plugins pre-assembled and archived in a tar on the build.
                    if (buildNumber.toLowerCase() == "latest") {
                        selector = LAST_COMPLETED_BUILD_SELSECTOR;
                    } else {
                        // Get from a specific build number. This run may have been triggered from a
                        // build of a Blue Ocean branch.
                        selector = [$class: 'SpecificBuildSelector', buildNumber: "${buildNumber}"];
                    }

                    // Let's copy and extract that tar to where the ATH would expect the plugins to be.
                    // Try checking out the Blue Ocean branch having the name supplied by build parameter. If that fails
                    // (i.e. doesn't exist ), just use the default/master branch and run the ATH tests against that.
                    try {
                        step ([$class: 'CopyArtifact',
                               projectName: "blueocean/${urlEncodedBranchName}",
                               selector: selector,
                               filter: 'blueocean/target/ath-plugins.tar.gz']);
                    } catch (Exception e) {
                        echo ""
                        echo "Error copying pre-assembled plugins tar from Blue Ocean branch named '${urlEncodedBranchName}'."
                        echo "  - Error message: ${e.getMessage()}"
                        echo "  - Build selector used: '${selector}'."
                        echo "  - Trying the last completed 'master' build instead..."
                        echo ""
                        branchName = "master";
                        buildNumber = DEFAULT_BUILD_NUM;
                        step ([$class: 'CopyArtifact',
                               projectName: "blueocean/master",
                               selector: LAST_COMPLETED_BUILD_SELSECTOR,
                               filter: 'blueocean/target/ath-plugins.tar.gz']);
                        echo ""
                        echo "Copying pre-assembled plugins tar from last completed 'master' build was successful."
                        echo ""
                    }
                    sh 'mkdir -p blueocean-plugin/blueocean'
                    sh 'tar xzf blueocean/target/ath-plugins.tar.gz -C blueocean-plugin/blueocean'
                    // Mark this as a pre-assembly. This tells the run.sh script to
                    // not perform the assembly again.
                    sh 'touch blueocean-plugin/blueocean/.pre-assembly'
                }
                sh "mvn -B clean install -DskipTests -s settings.xml"

                // Run the ATH. Tell the run script to not try starting selenium. Selenium is
                // already running in a docker container of it's on in the host. See call to
                // ./start-selenium.sh (above) and ./stop-selenium.sh (below).
                stage 'run'
                sh "./run.sh -a=./blueocean-plugin/blueocean/ --no-selenium"
            } catch (err) {
                currentBuild.result = "FAILURE"

                if (err.toString().contains('AbortException')) {
                    currentBuild.result = "ABORTED"
                }
            } finally {
                sendhipchat(repoUrl, branchName, buildNumber, null)
                step([$class: 'JUnitResultArchiver', testResults: 'target/surefire-reports/**/*.xml'])
            }
        }
        } // configFileProvider
    } finally {
        sh "./stop-selenium.sh"
    }
}

def sendhipchat(repoUrl, branchName, buildNumber, err) {
    def res = currentBuild.result
    if(res == null) {
        res = "SUCCESS"
    }

    def shortRepoURL = toShortRepoURL(repoUrl);
    def repoBranchURL = toRepoBranchURL(repoUrl, branchName);
    message = "ATH: <a href='${env.RUN_DISPLAY_URL}'>${env.JOB_NAME} #${env.BUILD_NUMBER}</a><br/>"
    message += "- run against: <a href='${repoBranchURL}'>${shortRepoURL}:${branchName}</a>"
    if (buildNumber == '') {
        message += ' (HPIs built from branch source)<br/>'
    } else {
        message += " (HPIs from build #${buildNumber})<br/>"
    }
    if (err == null) {
        message += "- result: ${res}"
    } else {
        message += "- result: ${res}<br/>"
        message += "<pre>${drainStacktrace(err)}</pre>"
    }

    def color = null
    if(res == "UNSTABLE") {
        color = "YELLOW"
    } else if(res == "SUCCESS"){
        color = "GREEN"
    } else if(res == "FAILURE") {
        color = "RED"
    } else if(res == "ABORTED") {
        color = "GRAY"
    }
    if(color != null) {
        hipchatSend message: message, color: color
    }
}

def toShortRepoURL(repoURL) {
    def parsedReproUri = new URI(repoURL)
    def repoPath = parsedReproUri.getPath();

    if (repoPath.startsWith("/")) {
        repoPath = repoPath.substring(1);
    }
    repoPath = repoPath.replace('.git', '');

    return repoPath;
}

def toRepoBranchURL(repoURL, branchName) {
    def repoBranchURL = repoURL;

    repoBranchURL = repoBranchURL.replace('.git', '');
    repoBranchURL += '/tree/' + branchName;

    return repoBranchURL;
}

def drainStacktrace(throwable) {
    def stringWriter = new StringWriter();
    throwable.printStackTrace(new PrintWriter(stringWriter));
    def string = stringWriter.toString()
    if (string.length() > 650) {
        string = string.substring(0, 650) + ' (truncated) ...';
    }
    return string;
}
