pipeline {
  agent {
    node {
      label 'nomad'
    }
    
  }
  stages {
    stage('Prepare Environment') {
      steps {
        sh '''#!/bin/bash
source /etc/profile.d/rvm.sh
rvm use 2.1.9
which rake || gem install rake
which bundle || gem install bundler
bundle install'''
      }
    }
    stage('Build') {
      steps {
        sh '''#!/bin/bash
source /etc/profile.d/rvm.sh
bundle exec jekyll build'''
      }
    }
  }
}