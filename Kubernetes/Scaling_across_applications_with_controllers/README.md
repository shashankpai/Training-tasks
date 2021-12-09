# Deployment Controller

Deployments are probably the resource you’ll use most in Kubernetes, and you’ve already had lots of experience with them. Now it’s time to dig a bit deeper and learn that Deployments don’t actually manage Pods directly—that’s done by another resource called a ReplicaSet.


You’ll use a Deployment to describe your app in most cases; the Deployment is a controller that manages ReplicaSets, and the ReplicaSet is a controller that manages Pods. You can create a ReplicaSet directly rather than using a Deployment

