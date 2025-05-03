# Creating new user
To create a new user, a folder needs to be created that automatically creates the user with all necessary access rights.
The user will be added to the `local` cluster and a new namespace will be created for the user.
In this namespace, the user will have full access to all resources.

1. Copy the `username` folder under `users` and rename it to the new username.
2. Change the username `<username>` in all files to the new username. This includes the `username` and `namespace`.
3. Change the password in the `user.yaml`, if you want. The password is stored as a Bcrypt hash with 10 rounds in the file. The standard password is `awesome`. Currently the passwords needs to be 12 characters long.
5. Push the changes to the repository master branch.
6. Login to Rancher with the new username and password. (ml-cluster.domaine.de)
    1. Go to the `local` cluster.
    2. On the upper right, click on the file icon for `Download KubeConfig`
7. Hand over the `kubeconfig` file to the new user so that he can save it at his computer under `~/.kube/configs`.