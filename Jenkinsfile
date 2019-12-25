
Map matrix_axes = [
    os: ['rhel7', 'centos7', 'rhel8', 'centos8'],
    hosttype: ['clip'],
    target_name: ['minimal', 'sftp-dropbox', 'apache', 'vpn'],
    media_type: ['inst-iso', 'live-iso'],
    os_version: ['7.6']
]

@NonCPS
List getMatrixAxes(Map matrix_axes) {
    List axes = []
    Map emptyMap = [:]
    List prevCombo = [emptyMap]
    for (entry in matrix_axes) {
        String axis = entry.key
        List values = entry.value
        List comboNext = []
        for (value in values) {
            for(int i = 0; i < prevCombo.size(); i++) {
                Map newMap = prevCombo[i].clone()
                newMap[axis] = value
                comboNext << newMap
            }
        }
        prevCombo = comboNext
    }
    return prevCombo
}

List axes = getMatrixAxes(matrix_axes)

@NonCPS
List getEnvList(Map axis) {
    List envList = []
    for (entry in axis) {
        envList << "${entry.key}=${entry.value}"
    }
    return envList
}

List getTaskMap(List axes) {
    List tasks = []
    for(int i = 0; i < axes.size(); i++) {
        Map axis = axes[i]
        List axisEnv = getEnvList(axis)
        String nodeLabel = axis['os'] + " && " + axis['hosttype']
        println("nodelabel: " + nodeLabel)
        String outerStage = "Build and test " + axis['os'] + "-" + axis['os_version'] + " " + axis['target_name'] + " " + axis['media_type']
        String prepareStage = "Prepare " + axis['os'] + "-" + axis['os_version'] + " " + axis['target_name'] + " " + axis['media_type']
        String buildStage = "Build " + axis['os'] + "-" + axis['os_version'] + " " + axis['target_name'] + " " + axis['media_type']
        String testStage = "Test " + axis['os'] + "-" + axis['os_version'] + " " + axis['target_name'] + " " + axis['media_type']
        String repoFile = "CONFIG_REPOS." + axis['os'] + "-" + axis['os_version']
        String target_name = axis['target_name']
        String media_type = axis['media_type']
        Map task = [name:axisEnv.join(', '), job:{ ->
            stage outerStage
            catchError(message:'stage failed', buildResult:'UNSTABLE', stageResult:'UNSTABLE') {
                // see if a particular node exists.  there may be better ways to do this
                try {
                    timeout(time: 2, unit: 'SECONDS') {
                        node(nodeLabel) {
                            sh "true"
                        }
                    }
                } catch (hudson.AbortException err) {
                    error("No suitable nodes found: ${err}")
                }
                try {
                    timeout(time: 120, unit: 'MINUTES') {
                        node(nodeLabel) {
                            sh "rm -rf .* *"
                            checkout scm
                            if(!fileExists(repoFile)) {
                                error("No CONFIG_REPO file " + repoFile)
                            }
                            sh "cp ${repoFile} CONFIG_REPOS"
                            sh "make clip-${target_name}-${media_type}"
                            archiveArtifacts "*.iso"
                            sh "./support/tests/media/qemu-test-${media_type}.sh *.iso"
                            archiveArtifacts(artifacts:"scap", allowEmptyArchive:true)
                        }
                    }
                } catch (hudson.AbortException err) {
                    error("Timeout exceeded during build and test")
                }
            }
        }]
        tasks << task
    }
    return tasks
}

List tasks = getTaskMap(axes)


for (int i=0; i< tasks.size(); i++) {
tasks[i]['job'].call()
}
