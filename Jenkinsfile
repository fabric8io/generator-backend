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
@Library('github.com/rawlingsj/fabric8-pipeline-library@master')
def releaseVersion
def newRelease
def name = 'generator-backend'

node{
  properties([
    parameters ([
            choice(choices: 'new release\nredeploy latest', description: 'Optionally avoid a new release and redeploy the latest available version?', name: 'release')
      ])
  ])
  newRelease = params.release == 'new release' ? true : false
}

if (newRelease){
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
    }
  }
} else {
  node {
    def cmd = "curl -L http://central.maven.org/maven2/io/fabric8/${name}/maven-metadata.xml | grep '<latest' | cut -f2 -d'>'|cut -f1 -d'<'"
    releaseVersion = sh(script: cmd, returnStdout: true).toString().trim()
    echo "Skipping release and redeploying ${releaseVersion}"
  }
}

deployOpenShiftNode(openshiftConfigSecretName: 'dsaas-preview-fabric8-forge-config', routeCertsSecretName: 'dsaas-preview-route-certs'){
  ws{
    stage "Deploying ${releaseVersion}"
    container(name: 'clients') {
      
      def prj = 'dsaas-preview-fabric8-forge'
      def forgeURL = 'forge.api.prod-preview.openshift.io'
      def yaml = "http://central.maven.org/maven2/io/fabric8/${name}/${releaseVersion}/${name}-${releaseVersion}-openshift.yml"

      def caCertificate = ''
      def certificate = ''
      def key = ''

      if (fileExists('/root/home/.routeCerts/caCertificate')){
        caCertificate = readFile '/root/home/.routeCerts/caCertificate'
        certificate = readFile '/root/home/.routeCerts/certificate'
        key = readFile '/root/home/.routeCerts/key'
      }

      wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [
          [password: caCertificate, var: 'ROUTE_CA_CERT'],
          [password: certificate, var: 'ROUTE_CERT'],
          [password: key, var: 'ROUTE_KEY']]]) {

        echo "now deploying to namespace ${prj}"
        sh """
          oc process -n ${prj} -f ${yaml} -v FORGE_URL=${forgeURL} -v CERTIFICATE=${certificate} -v CA_CERTIFICATE=${caCertificate} -v KEY=${key} | oc apply -n ${prj} -f -
        """
      }

      sleep 10 // ok bad bad but there's a delay between DC's being applied and new pods being started.  lets find a better way to do this looking at the new DC perhaps?

      waitUntil{
        // wait until the pods are running
        try{
          sh "oc get pod -l project=${name},provider=fabric8 -n ${prj} | grep Running"
          echo "${name} pod Running for v ${releaseVersion}"
          return true
        } catch (err) {
          echo "waiting for ${name} to be ready..."
          return false
        }
      }
    }
  }
}