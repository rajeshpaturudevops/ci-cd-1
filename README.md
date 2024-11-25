1. Jenkins Deployment
yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-deployment
  labels:
    app: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080 # Jenkins UI
        - containerPort: 50000 # Jenkins agent communication
        volumeMounts:
        - name: jenkins-data
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-data
        emptyDir: {} # Persistent storage should be configured in production
2. Jenkins Service
yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  labels:
    app: jenkins
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    targetPort: 8080
  - port: 50000
    targetPort: 50000
  selector:
    app: jenkins
3. Tomcat Deployment
yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  labels:
    app: tomcat
spec:
  replicas: 2 # Two Tomcat instances
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
      - name: tomcat
        image: tomcat:9
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: tomcat-webapps
          mountPath: /usr/local/tomcat/webapps
      volumes:
      - name: tomcat-webapps
        emptyDir: {} # Persistent storage should be configured for production
4. Tomcat Service
yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  labels:
    app: tomcat
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: tomcat
5. ConfigMap for Jenkins Pipeline
The Jenkins Pipeline will deploy the WAR file to both Tomcat servers.

yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jenkins-pipeline-config
data:
  Jenkinsfile: |
    pipeline {
      agent any
      stages {
        stage('Build') {
          steps {
            echo 'Building application...'
            // Add build steps here (e.g., Maven or Gradle)
          }
        }
        stage('Deploy to Tomcat') {
          steps {
            echo 'Deploying to Tomcat...'
            sh '''
              for ip in $(kubectl get pods -l app=tomcat -o jsonpath='{.items[*].status.podIP}'); do
                curl -T target/your-app.war http://$ip:8080/manager/text/deploy?path=/your-app&update=true \
                  --user admin:password;
              done
            '''
          }
        }
      }
    }
6. Persistent Volume for Jenkins (Optional)
For production use, add persistent storage for Jenkins to save build data.

yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
Update the Jenkins deployment to use this PVC:

yaml
volumes:
- name: jenkins-data
  persistentVolumeClaim:
    claimName: jenkins-pv
