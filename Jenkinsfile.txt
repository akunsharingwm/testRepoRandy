pipeline {
  agent any

  environment {
    SAG_HOME       = '/opt/wm1015'
    ABE_BIN        = "${env.SAG_HOME}/common/AssetBuildEnvironment/bin"
    DEPLOYER_BIN   = "${env.SAG_HOME}/IntegrationServer/instances/default/packages/WmDeployer/bin"
    REPO_ROOT      = '/opt/wm1015/asset-package/output'
    TARGET_GROUP   = 'IS_Package'
    DEPLOYER_HOST  = '127.0.0.1'
    DEPLOYER_PORT  = '5555'
  }

  options { timestamps() }

  stages {

    stage('Build ABE → Repo') {
      steps {
        sh '''
          set -euo pipefail
          echo "== Build ABE dari $WORKSPACE =="
          cd "$ABE_BIN"
          ./build.sh -Dbuild.source.dir="$WORKSPACE"

          echo "== Isi repo setelah build =="
          ls -al "$REPO_ROOT/IS" || true
        '''
      }
    }

    stage('Generate Names & ProjectAutomator') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'webMethodDeployer',
                                          usernameVariable: 'WM_USER',
                                          passwordVariable: 'WM_PWD')]) {
          sh '''
            set -euo pipefail
            PKG_ACDL=$(find "$REPO_ROOT/IS" -type f -name '*.acdl' -printf '%T@ %p\\n' | sort -n | tail -1 | awk '{print $2}')
            if [ -z "${PKG_ACDL:-}" ]; then
              PKG_ZIP=$(find "$REPO_ROOT/IS" -type f -name '*.zip' -printf '%T@ %p\\n' | sort -n | tail -1 | awk '{print $2}')
              [ -z "${PKG_ZIP:-}" ] && { echo "Tidak menemukan *.acdl / *.zip di $REPO_ROOT/IS"; exit 1; }
              PACKAGE_NAME="$(basename "$PKG_ZIP" .zip)"
            else
              PACKAGE_NAME="$(basename "$PKG_ACDL" .acdl)"
            fi

            # --- Dynamic Name ---#
            PROJECT_NAME="PACKAGE_${PACKAGE_NAME}_PROJECT"
            SET_NAME="${PACKAGE_NAME}-set-${BUILD_NUMBER}"
            MAP_NAME="${PACKAGE_NAME}-map-${BUILD_NUMBER}"
            DC_NAME="${PACKAGE_NAME}-dc-${BUILD_NUMBER}"

            echo "PACKAGE_NAME : $PACKAGE_NAME"
            echo "PROJECT_NAME : $PROJECT_NAME"
            echo "SET_NAME     : $SET_NAME"
            echo "MAP_NAME     : $MAP_NAME"
            echo "DC_NAME      : $DC_NAME"

            {
              echo "PACKAGE_NAME=$PACKAGE_NAME"
              echo "PROJECT_NAME=$PROJECT_NAME"
              echo "SET_NAME=$SET_NAME"
              echo "MAP_NAME=$MAP_NAME"
              echo "DC_NAME=$DC_NAME"
            } > "$WORKSPACE/.deploy_env"

            # --- DeployerSpec: (Composite name=PACKAGE_NAME) ---
            SPEC="$WORKSPACE/deployer-spec.xml"
            cat > "$SPEC" <<XML
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<DeployerSpec exitOnError="true" sourceType="Repository">
  <DeployerServer>
    <host>${DEPLOYER_HOST}:${DEPLOYER_PORT}</host>
    <user>${WM_USER}</user>
    <pwd>${WM_PWD}</pwd>
  </DeployerServer>

  <Environment>
    <Repository>
      <repalias name="repo_IS">
        <type>FlatFile</type>
        <urlOrDirectory>${REPO_ROOT}</urlOrDirectory>
        <Test>true</Test>
      </repalias>
    </Repository>
  </Environment>

  <Projects projectPrefix="">
    <Project description="" name="${PROJECT_NAME}" overwrite="true" type="Repository">
      <ProjectProperties>
        <Property name="projectLocking">false</Property>
        <Property name="concurrentDeployment">true</Property>
        <Property name="ignoreMissingDependencies">false</Property>
        <Property name="isTransactionalDeployment">false</Property>
      </ProjectProperties>

      <DeploymentSet autoResolve="full" description="" name="${SET_NAME}" srcAlias="repo_IS">
        <!-- HANYA package yang mau dideploy -->
        <Composite name="${PACKAGE_NAME}" srcAlias="repo_IS" type="IS"/>
      </DeploymentSet>

      <DeploymentMap description="" name="${MAP_NAME}"/>
      <MapSetMapping mapName="${MAP_NAME}" setName="${SET_NAME}">
        <group type="IS">${TARGET_GROUP}</group>
      </MapSetMapping>

      <DeploymentCandidate description="" mapName="${MAP_NAME}" name="${DC_NAME}"/>
    </Project>
  </Projects>
</DeployerSpec>
XML

            echo "== Jalankan Project Automator =="
            /bin/bash "$DEPLOYER_BIN/projectautomatorUnix.sh" "$SPEC"
          '''
        }
      }
    }

    stage('Checkpoint') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'webMethodDeployer',
                                          usernameVariable: 'WM_USER',
                                          passwordVariable: 'WM_PWD')]) {
          sh '''
            set -euo pipefail
            . "$WORKSPACE/.deploy_env"

            echo "== CHECKPOINT DC: $DC_NAME =="
            "$DEPLOYER_BIN/Deployer.sh" --checkpoint \
              -dc "$DC_NAME" -project "$PROJECT_NAME" \
              -host "$DEPLOYER_HOST" -port "$DEPLOYER_PORT" \
              -user "$WM_USER" -pwd "$WM_PWD" \
              -reportFilePath /opt/wm1015/deployer-reports
          '''
        }
      }
    }

    stage('Deploy') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'webMethodDeployer',
                                          usernameVariable: 'WM_USER',
                                          passwordVariable: 'WM_PWD')]) {
          sh '''
            set -euo pipefail
            . "$WORKSPACE/.deploy_env"

            echo "== DEPLOY DC: $DC_NAME =="
            "$DEPLOYER_BIN/Deployer.sh" --deploy \
              -dc "$DC_NAME" -project "$PROJECT_NAME" \
              -host "$DEPLOYER_HOST" -port "$DEPLOYER_PORT" \
              -user "$WM_USER" -pwd "$WM_PWD" \
              -reportFilePath /opt/wm1015/deployer-reports
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'deployer-reports/**/*', allowEmptyArchive: true
    }
  }
}
