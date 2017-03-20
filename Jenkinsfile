#!/usr/bin/groovy
/**
 * Copyright (C) 2015 Red Hat, Inc.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *         http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
@Library('github.com/fabric8io/fabric8-pipeline-library@master')
def dummy
deployOpenShiftTemplate(openshiftConfigSecretName: 'dsaas-preview-fabric8-forge-config'){
  releaseNode{
    ws{
      checkout scm
      readTrusted 'release.groovy'
      sh "git remote set-url origin git@github.com:fabric8io/generator-backend.git"

      def pipeline = load 'release.groovy'

      stage 'Stage'
      def stagedProject = pipeline.stage()
      releaseVersion = stagedProject[1]

      stage 'Promote'
      pipeline.release(stagedProject)

      container(name: 'clients') {
        stage "Applying ${releaseVersion} updates"
        def prj = 'dsaas-preview-fabric8-forge'
        def name 'generator-backend'
        def forgeURL = 'forge.api.prod-preview.openshift.io'
        sh """

          echo "now deploying to namespace ${prj}"
          oc process -n ${prj} -f http://central.maven.org/maven2/io/fabric8/${name}/${releaseVersion}/${name}-${releaseVersion}-openshift.yml -v FORGE_HOST=${forgeURL} | oc apply -n ${prj} -f -

          sleep 10 // ok bad bad but there's a delay between DC's being applied and new pods being started.  lets find a better way to do this looking at the new DC perhaps?

          waitUntil{
            // wait until the pods are running has been deleted
            try{
              sh "oc get pod -l project=${name},provider=fabric8 -n ${prj} | grep Running"
              echo "${name} pod Running for v ${releaseVersion}"
              return true
            } catch (err) {
              echo "waiting for ${name} to be ready..."
              return false
            }
          }
        """
      }
    }
  }
}