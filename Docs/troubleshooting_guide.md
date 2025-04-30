# Troubleshooting Guide: EC2 Deployment & User Data Issues

This guide covers common issues encountered when deploying applications to EC2 using Terraform `user_data` scripts and CI/CD pipelines (like GitHub Actions) that interact with the instance via SSH.

## Issue 1: `user_data` Script Fails to Execute or Complete

**Symptoms:**
*   Tools expected to be installed by `user_data` (Docker, AWS CLI, Nginx, etc.) are missing when you SSH into the instance (`command not found`).
*   Configuration expected from `user_data` (e.g., Nginx config files, generated certificates) is missing.

**Diagnosis:**
This means the `user_data` script itself failed during the instance's initial boot process.

**Debugging Steps:**

1.  **Check `user-data.log`:** The first place to look is the log file we configured the script to write to. SSH into the instance and run:
    ```bash
    cat /var/log/user-data.log
    ```
    Look for any error messages near the end of the log. If this file *doesn't exist*, it indicates the script failed very early, possibly before the logging redirection took effect.

2.  **Check `cloud-init-output.log`:** This log captures general output from the cloud-init process, including messages about script execution failures.
    ```bash
    cat /var/log/cloud-init-output.log
    ```
    Look for lines like `Failed to run module scripts_user` or `Running module scripts_user ... failed`.

3.  **Check `cloud-init.log`:** This is the most detailed cloud-init log and often contains the specific error message from within the script or the reason cloud-init couldn't execute it.
    ```bash
    cat /var/log/cloud-init.log
    ```
    Search this log for "ERROR", "WARNING", "FAIL", or the name of the script file (e.g., `part-001`).

**Common Causes & Solutions:**

1.  **Exec Format Error / Missing Shebang:**
    *   **Cause:** The `cloud-init.log` shows `Exec format error. Missing #! in script?`. The operating system doesn't recognize the `user_data` script as executable because the very first line isn't `#!/bin/bash` (or another valid shebang). This can happen due to incorrect line endings (Windows CRLF vs. Linux LF) or hidden characters before the shebang.
    *   **Solution:** Rewrite the `user_data` block within your Terraform file (`aws_instance` resource) using the `replace_in_file` tool or manually ensuring the `#!/bin/bash` line is absolutely first and uses correct Linux line endings. Re-run `terraform apply` (this will replace the instance).

2.  **Command Errors within Script:**
    *   **Cause:** One of the commands inside your `user_data` script failed (e.g., `apt-get install` failed due to network issues or a typo, a configuration command failed). Since `set -e` is usually active, the script stops execution at the first error.
    *   **Solution:** Identify the failing command in the logs (`user-data.log` or `cloud-init.log`). Correct the command in your Terraform `user_data` block and re-run `terraform apply`.

3.  **Permissions Issues:**
    *   **Cause:** The script tries to write to a directory or execute something it doesn't have permission for (though `user_data` usually runs as root).
    *   **Solution:** Adjust commands or permissions as needed (e.g., use `sudo` if necessary, though often not required for `user_data`).

## General Troubleshooting Tips

*   **Security Groups:** Always verify that the EC2 instance's security group allows inbound traffic on the necessary ports (e.g., 22 for SSH, 80/443 for HTTP/S) from the correct source IPs.
*   **Service Status:** After deployment (and waiting for `user_data`), SSH in and check the status of key services: `sudo systemctl status docker`, `sudo systemctl status nginx`.
*   **Logs:** Check application logs (`docker logs <container_name>`) and web server logs (`/var/log/nginx/error.log`, `/var/log/nginx/access.log`).
*   **Terraform Plan:** Always run `terraform plan` before `terraform apply` to understand exactly what changes will be made, especially looking for resource replacements (`-/+`).

## Issue 2: `terraform plan` Doesn't Show EC2 Instance Replacement After `user_data` Change

**Symptoms:**
*   You have modified the `user_data` script within an `aws_instance` resource block in your Terraform code.
*   You expect `terraform plan` to show the instance being replaced (`-/+ destroy and then create replacement`).
*   Instead, the plan shows "0 to destroy" or only indicates in-place changes (`~`) for other resources (like security groups).

**Diagnosis:**
This usually happens because Terraform determines the need for replacement based on comparing the *hash* of the current `user_data` content with the hash stored in the Terraform state file from the **last successful `terraform apply`**. If the last successful `apply` didn't include the `user_data` change (perhaps because a previous `plan` or `apply` failed after the code change but before the state was updated), or if the hash comparison logic doesn't detect a difference against the *last applied state*, Terraform might incorrectly conclude that no replacement is needed. However, since `user_data` only runs on the first boot, the *currently running* instance will not have the new configuration.

**Solution: Force Replacement with `terraform taint`**

To ensure the new `user_data` script is executed on a fresh instance, you need to explicitly tell Terraform to replace the existing one using the `terraform taint` command.

1.  **Identify the Resource Address:** Find the full address of the EC2 instance resource in your Terraform code. For modules, it usually follows the pattern `module.<module_instance_name>.<resource_type>.<resource_name>`. In this project, it's likely `module.aws_compute.aws_instance.app_server`.
2.  **Run `terraform taint`:** Execute the command in the directory where you run `plan`/`apply` (e.g., `environments/dev/`):
    ```bash
    terraform taint module.aws_compute.aws_instance.app_server
    ```
    *(Replace the address if your module/resource names are different)*
3.  **Verify with `plan`:** Run `terraform plan` again. It should now clearly show the tainted instance marked for replacement (`-/+ destroy and then create replacement`).
4.  **Apply:** Run `terraform apply`. Terraform will destroy the old (tainted) instance and create a new one, which will execute the latest `user_data` script on its first boot.

**Note:** Tainting is a way to manually override Terraform's state comparison when you know a resource needs to be recreated even if Terraform doesn't automatically detect it based on configuration changes alone.
