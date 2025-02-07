---
description: Set up a systemd service to daemonize a Prefect worker or create a long-running deployment serve process
tags:
    - systemd
    - daemonize
    - worker
search:
  boost: 2
---

# Daemonize Processes for Prefect Deployments

When running workflow applications, it can be helpful to create long-running processes that run at startup and are robust to failure.
In this guide you'll learn how to set up a systemd service to create long-running Prefect processes that poll for scheduled flow runs.

A systemd service is ideal for running a long-lived process on a Linux VM or physical Linux server.
We will leverage systemd and see how to automatically start a [Prefect worker](/concepts/work-pools/#worker-overview) or long-lived [`serve` process](concepts/flows/#serving-a-flow) when Linux starts.
This approach provides resilience by automatically restarting the process if it crashes.

In this guide we will:

* Create a Linux user
* Install and configure Prefect
* Set up a systemd service for the Prefect worker or `.serve` process

## Prerequisites

* An environment with a linux operating system with [systemd](https://systemd.io/) and Python 3.8 or later.
* A superuser account (you can run `sudo` commands).
* A Prefect Cloud account, or a local instance of a Prefect server running on your network.
* If daemonizing a worker, you'll need a Prefect [deployment](/concepts/deployments/) with a [work pool](/concepts/work-pools/) your worker can connect to.

If using an [AWS t2-micro EC2 instance](https://aws.amazon.com/ec2/instance-types/t2/) with an AWS Linux image, you can install Python and pip with `sudo yum install -y python3 python3-pip`.

## Step 1: Add a user

Create a user account on your linux system for the Prefect process.
While you can run a worker or serve process as root, it's good security practice to avoid doing so unless you are sure you need to.

In a terminal, run:

```bash
sudo useradd -m prefect
sudo passwd prefect
```

When prompted, enter a password for the `prefect` account.

Next, log in to the `prefect` account by running:

```bash
sudo su prefect
```

## Step 2: Install Prefect

Run:

```bash
pip3 install prefect
```

This guide assumes you are installing Prefect globally, not in a virtual environment.
If running a systemd service in a virtual environment, you'll just need to change the ExecPath.
For example, if using [venv](https://docs.python.org/3/library/venv.html), change the ExecPath to target the `prefect` application in the `bin` subdirectory of your virtual environment.

Next, set up your environment so that the Prefect client will know which server to connect to.

If connecting to Prefect Cloud, follow [the instructions](https://docs.prefect.io/ui/cloud-getting-started/#create-an-api-key) to obtain an API key and then run the following:

```bash
prefect cloud login -k YOUR_API_KEY
```

When prompted, choose the Prefect workspace you'd like to log in to.

If connecting to a self-hosted Prefect server instance instead of Prefect Cloud, run the following and substitute the IP address of your server:

```bash
prefect config set PREFECT_API_URL=http://your-prefect-server-IP:4200
```

Finally, run the `exit` command to sign out of the `prefect` Linux account.
This command switches you back to your sudo-enabled account so you will can run the commands in the next section.

## Step 3: Set up a systemd service

=== "Prefect worker"

    Move into the `/etc/systemd/system` folder and open a file for editing.
    We use the Vim text editor below.

    ```bash
    cd /etc/systemd/system
    sudo vim my-prefect-service.service
    ```

    ```title="my-prefect-service.service"
    [Unit]
    Description=Prefect worker

    [Service]
    User=prefect
    WorkingDirectory=/home
    ExecStart=prefect worker start --pool YOUR_WORK_POOL_NAME
    Restart=always

    [Install]
    WantedBy=multi-user.target
    ```

    Make sure you substitute your own work pool name.

=== "`.serve`"

    Copy your flow entrypoint Python file and any other files needed for your flow to run into the `/home` directory (or the directory of your choice).

    Here's a basic example flow:

    ```python title="my_file.py"
    from prefect import flow


    @flow(log_prints=True)
    def say_hi():
        print("Hello!")
        
    if __name__=="__main__":
        say_hi.serve(name="Greeting from daemonized .serve")
    ```

    If you want to make changes to your flow code without restarting your process, you can push your code to git-based cloud storage (GitHub, BitBucket, GitLab) and use `flow.from_source().serve()`, as in the example below.

    ```python title="my_remote_flow_code_file.py"
    if __name__ == "__main__":
      flow.from_source(
          source="https://github.com/org/repo.git",
          entrypoint="path/to/my_remote_flow_code_file.py:say_hi",
      ).serve(name="deployment-with-github-storage")
    ```

    Make sure you substitute your own flow code entrypoint path.

    Note that if you change the flow entrypoint parameters, you will need to restart the process.

    Move into the `/etc/systemd/system` folder and open a file for editing.
    We use the Vim text editor below.

    ```bash
    cd /etc/systemd/system
    sudo vim my-prefect-service.service
    ```

    ```title="my-prefect-service.service"
    [Unit]
    Description=Prefect serve 

    [Service]
    User=prefect
    WorkingDirectory=/home
    ExecStart=python3 my_file.py
    Restart=always

    [Install]
    WantedBy=multi-user.target
    ```

To save the file and exit Vim hit the escape key, type `:wq!`, then press the return key.

Next, run `sudo systemctl daemon-reload` to make systemd aware of your new service.

Then, run `sudo systemctl enable my-prefect-service` to enable the service.
This command will ensure it runs when your system boots.

Next, run `sudo systemctl start my-prefect-service` to start the service.

Run your deployment from UI and check out the logs on the **Flow Runs** page.

You can see if your daemonized Prefect worker or serve process is running and see the Prefect logs with `systemctl status my-prefect-service`.

That's it!
You now have a systemd service that starts when your system boots, and will restart if it ever crashes.

## Next steps

If you want to set up a long-lived process on a Windows machine the pattern is similar.
Instead of systemd, you can use [NSSM](https://nssm.cc/).

Check out other [Prefect guides](/guides/) to see what else you can do with Prefect!
