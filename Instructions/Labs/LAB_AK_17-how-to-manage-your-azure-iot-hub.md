---
lab:
    title: 'Lab 17: How to manage your Azure IoT Hub'
    module: 'Module 9: Solution Testing, Diagnostics, and Logging'
---

# How to manage your Azure IoT Hub

## Lab Scenario

Contoso's Asset Monitoring and Tracking Solution is working great. The system provides continuous monitoring throughout the packaging and shipping process. You have implemented Group Enrollments within DPS to provision devices at scale, and when the container arrives at the destination, the IoT device "decommissioned" through DPS so that it can re-used for future shipments.

To help manage device utilization and other characteristics of the solution, the IT department has asked your team to implement Azure monitoring and logging services within the IoT solution.

You agree to begin by implementing some simple metrics that can be reviewed with the IT folks before you commit to any additional workload.

In this lab, you will implement monitoring to track the number of connected devices and telemetry messages sent, as well as send connection events to a log. In addition, you will create an alert that will be triggered based upon the average number of devices connected. To test the system, you will configure 10 simulated IoT Devices that will authenticate with DPS using a Device CA Certificate generated on the Root CA Certificate chain. The IoT Devices will be configured to send telemetry to the the IoT Hub.

## In This Lab

In this lab, you will complete the following activities:

* Verify Lab Prerequisites
* Enable diagnostic logs.
* Enable metrics.
* Set up alerts for those metrics.
* Download and run an app that simulates IoT devices connecting via X.509 and sending messages to the hub.
* Run the app until the alerts begin to fire.
* View the metrics results and check the diagnostic logs.

## Lab Instructions

### Exercise 1: Verify Lab Prerequisites

This lab assumes that the following Azure resources are available:

| Resource Type | Resource Name |
| :-- | :-- |
| Resource Group | AZ-220-RG |
| IoT Hub | AZ-220-HUB-_{YOUR-ID}_ |
| Device Provisioning Service | AZ-220-DPS-_{YOUR-ID}_ |
| Storage Account | AZ-220-STORAGE-_{YOUR-ID}_ |

If these resources are not available, you will need to run the **lab17-setup.azcli** script as instructed below before moving on to Exercise 2. The script file is included in the GitHub repository that you cloned locally as part of the dev environment configuration (lab 3).

The **lab17-setup.azcli** script is written to run in a **bash** shell environment - the easiest way to execute this is in the Azure Cloud Shell.

1. Using a browser, open the [Azure Cloud Shell](https://shell.azure.com/) and login with the Azure subscription you are using for this course.

    If you are prompted about setting up storage for Cloud Shell, accept the defaults.

1. Verify that the Azure Cloud Shell is using **Bash**.

1. On the Azure Shell toolbar, click **Upload/Download files** (fourth button from the right).

1. In the dropdown, click **Upload**.

1. In the file selection dialog, navigate to the folder location of the GitHub lab files that you downloaded when you configured your development environment.

    In _Lab 3: Setup the Development Environment_, you cloned the GitHub repository containing lab resources by downloading a ZIP file and extracting the contents locally. The extracted folder structure includes the following folder path:

    * Allfiles
      * Labs
          * 17-How to manage your Azure IoT Hub
            * Setup

    The lab17-setup.azcli script file is located in the Setup folder for lab 17.

1. Select the **lab17-setup.azcli** file, and then click **Open**.

    A notification will appear when the file upload has completed.

1. To verify that the correct file has uploaded in Azure Cloud Shell, enter the following command:

    ```bash
    ls
    ```

    The `ls` command lists the content of the current directory. You should see the lab17-setup.azcli file listed.

1. To create a directory for this lab that contains the setup script and then move into that directory, enter the following Bash commands:

    ```bash
    mkdir lab17
    mv lab17-setup.azcli lab17
    cd lab17
    ```

1. To ensure the **lab17-setup.azcli** script has the execute permission, enter the following command:

    ```bash
    chmod +x lab17-setup.azcli
    ```

1. On the Cloud Shell toolbar, to edit the lab17-setup.azcli file, click **Open Editor** (second button from the right - **{ }**).

1. In the **Files** list, to expand the lab17 folder and open the script file, click **lab17**, and then click **lab17-setup.azcli**.

    The editor will now show the contents of the **lab17-setup.azcli** file.

1. In the editor, update the values of the `{YOUR-ID}` and `{YOUR-LOCATION}` variables.

    Referencing the sample below as an example, you need to set `{YOUR-ID}` to the Unique ID you created at the start of this course - i.e. **CAH191211**, and set `{YOUR-LOCATION}` to the location that makes sense for your resources.

    ```bash
    #!/bin/bash

    YourID="{YOUR-ID}"
    RGName="AZ-220-RG"
    IoTHubName="AZ-220-HUB-$YourID"
    DPSName="AZ-220-DPS-$YourID"
    DeviceName="asset-track"
    Location="{YOUR-LOCATION}"
    ```

    > **Note**:  The `{YOUR-LOCATION}` variable should be set to the short name for the region. You can see a list of the available regions and their short-names (the **Name** column) by entering this command:
    >
    > ```bash
    > az account list-locations -o Table
    >
    > DisplayName           Latitude    Longitude    Name
    > --------------------  ----------  -----------  ------------------
    > East Asia             22.267      114.188      eastasia
    > Southeast Asia        1.283       103.833      southeastasia
    > Central US            41.5908     -93.6208     centralus
    > East US               37.3719     -79.8164     eastus
    > East US 2             36.6681     -78.3889     eastus2
    > ```

1. In the top-right of the editor window, to save the changes made to the file and close the editor, click **...**, and then click **Close Editor**.

    If prompted to save, click **Save** and the editor will close.

    > **Note**:  You can use **CTRL+S** to save at any time and **CTRL+Q** to close the editor.

1. To create the resources required for this lab, enter the following command:

    ```bash
    ./lab17-setup.azcli
    ```

    This script can take a few minutes to run. You will see JSON output as each step completes.

    The script will first create a resource group named **AZ-220-RG**, then your IoT Hub named **AZ-220-HUB-{YourID}** and Device Provisioning Service named **AZ-220-DSP-{YourID}**. If the services already exist, a corresponding message will be displayed. The script will link your IoT Hub and DSP. The script will then create a storage account named **az220storage{your-id}**.

    You should now be ready to proceed with Exercise 2 of this lab.

### Exercise 2: Set Up and Use Metrics and Diagnostic Logs with an IoT Hub

Azure Resource logs are platform logs emitted by Azure resources that describe their internal operation. All resource logs share a common top-level schema with the flexibility for each service to emit unique properties for their own events.

When you have an IoT Hub solution running in production, you will want to set up various metrics and enable diagnostic logs. Then, if a problem occurs, you have data to look at that will help you to diagnose the problem and fix it more quickly. 

In this exercise, you will enable diagnostic logs and use them to to check for errors. You will also set up some metrics to watch, and alerts that fire when the metrics hit a certain boundary.

#### Task 1: Enable diagnostics

1. Sign in to the **Azure portal** and navigate to your IoT hub.

1. In the left hand navigation menu, under **Monitoring**, click **Diagnostic settings**.

    > **Note**:  Diagnostics are disabled by default.

1. At the top of the **Diagnostic settings** page, under **Subscription**, select the subscription you used to create the IoT Hub.

1. Under **Resource group**, select the resource group you used for this lab - "AZ-220-RG".

1. Under **Resource type**, select **IoT Hub**.

1. Under **Resource**, select the IoT Hub you are using for this lab - **AZ-220-HUB-\<INITIALS-DATE\>**.

    Once you select the resource, the page will update with the option to turn on diagnostics, as well as a list of available metrics to monitor.

1. To turn on diagnostics, click **Turn on diagnostics**.

    The **Diagnostic settings** detail pane will be shown.

1. Under **Name**, enter **diags-hub**.

    You can see that there are 3 options available for routing the metrics - you can learn more about each by following the links below:

    * [Archive Azure resource logs to storage account](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/resource-logs-collect-storage)
    * [Stream Azure monitoring data to an event hub](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/stream-monitoring-data-event-hubs)
    * [Collect Azure resource logs in Log Analytics workspace in Azure Monitor](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/resource-logs-collect-workspace)

    In this lab we will use the storage account option.

1. Check **Archive to a storage account** and the **Storage account** configuration section will appear.

1. To specify the storage account to use, click **Configure**.

    The **Select a storage account** pane will appear.

    > **Note**:  In production, you should not use an existing storage account that has other, non-monitoring data stored in it so that you can better control access to monitoring data. If you are also archiving the Activity log to a storage account though, you may choose to use that same storage account to keep all monitoring data in a central location.

1. On the  **Select a storage account** pane, under **Subscription**, select the subscription you used to create the storage account earlier.

1. Under **Storage account**, select the storage account you created earlier.

1. To complete the storage account selection, click **OK**.

    The **Select a storage account** pane will close and the specified storage account will be displayed under **Storage account**.

1. Under **log**, check **Connections** and **Device Telemetry** and then update the **Retention (days)** value for each to **7**. You can do this by either moving the slider or directly entering **7** into the value textbox.

1. Click **Save** to save the settings.

1. Close the **Diagnostics settings** pane.

    The main **Diagnostics settings** page is displayed - you should see that the list of **Diagnostics settings** has now been updated to show the **diags-hub** setting you just created.

Later, when you look at the diagnostic logs, you'll be able to see the connect and disconnect logging for the device.

#### Task 2: Setup Metrics

Now set up some metrics to watch for when messages are sent to the hub.

1. In the left hand navigation area, under **Monitoring**, click **Metrics**.

    The **Metrics** pane is displayed showing a new, empty, chart.

1. To change the time range and granularity for the chart, at the top-right of the screen, click **Last 24 hours (Automatic)**.

1. In the dropdown that appears, select **Last 4 hours** for **Time Range**, and set **Time Granularity** to **1 minute**, and ensure **Show time as** is set to **local time**.

1. Click **Apply** to save these settings.

1. Under the **Chart Title** and toolbar, you will see a default metric entry.

    We will now add a metric to monitor how many telemetry messages have been sent.

1. Note that the **SCOPE** is already set to the IoT Hub.

1. Under **METRIC NAMESPACE**, note that the **IoT Hub standard metrics** namespace is selected.

    > **Note**:  By default, there is only one metric namespace available. Namespaces are a way to categorize or group similar metrics together. By using namespaces, you can achieve isolation between groups of metrics that might collect different insights or performance indicators. For example, you might have a namespace called **az220memorymetrics** that tracks memory-use metrics which profile your app. Another namespace called **az220apptransaction** might track all metrics about user transactions in your application. You can learn more about custom metrics and namespaces [here](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/metrics-custom-overview?toc=%2Fazure%2Fazure-monitor%2Ftoc.json#namespace).

1. In the **METRIC** dropdown list, select **Telemetry messages sent**. Notice how many metrics are available!

1. Under **AGGREGATION**, select **Sum**. Notice there are 4 aggregation operations available - *Avg*, *Min*, *Max* and *Sum*.

    We have completed the specification for the first metric. Notice that the chart title has updated to reflect the metric chosen. Now let's add another to monitor the total number of messages used.

1. Under the updated **Chart Title**, in the toolbar, click **Add metric**.

    A new metric will appear. Notice that, again, the **SCOPE** and **METRIC NAMESPACE** values are pre-populated and the **METRIC** dropdown is focused and open.

1. Under **METRIC**, select **Connected devices (preview)**.

1. Under **AGGREGATION**, select **Avg**.

    Your screen now shows the minimized metric for Telemetry messages sent, plus the new metric for avg connected devices. Notice that the chart title has updated to reflect both metrics.

    > **Note**:   To edit the chart title, click the **pencil** to the right of the title.

1. Under the updated **Chart Title**, in the toolbar, click **Pin to dashboard**. Note that you can choose to pin to the current dashboard or choose another. Select the dashboard you created in the first lab - "AZ-220-RG".

    > **Note**:  In order to retain the chart you have just created, it **must** be pinned to a dashboard.

1. Navigate to the "AZ-220-RG" dashboard and verify the chart is displayed.

    > **Note**:  You can customize the size and position of the chart by using drag and drop operations.

Now that we have enable logging and setup a chart to monitor metrics, we will set up an alert.

### Exercise 3: Configure an Alert

Now let us create an alert. Alerts proactively notify you when important conditions are found in your monitoring data. They allow you to identify and address issues before the users of your system notice them. In our asset tracking scenario, we use sensors to track our assets being transported. Each time a sensor is added in a transportation box, it will auto provision through DPS. We want to have a metric for the warehouse manager of how many boxes were "tagged" and need to count the Device Connected events from IoT Hub.

In this exercise, you are going to add an alert that will inform the warehouse manager when 5 or more devices have connected.

1. In the Azure Portal, navigate to the IoT Hub we are using for this lab.

1. In the left hand navigation area, under **Monitoring**, click **Alerts**.

    The empty **Alerts** page is displayed. Notice that the **Subscription**, **Resource group**, **Resource** and **Time range** fields are pre-populated.

1. Under **Time range**, select **Past hour**.

1. To add a new alert, click **+ New Alert Rule** (while the list is empty, you will see a **New Alert Rule** button in the center of the page - you can click this or the one in the toolbar).

    The **Create rule** pane is displayed.

1. At the top of the page, you will see two fields - **RESOURCE** and **HIERARCHY**. Notice they are pre-populated with the IoT Hub. To change the selected resource, you would click **Select**.

1. Under **Condition** you will see that no conditions have been defined. Click **Add** to add a new condition.

    The **Configure signal logic** pane is displayed. You will notice that there is a paginated table of available signals displayed. The fields above the table filter the table to assist in finding the signal types you want.

1. Under **Signal type**, you will note that **All** is selected. Click on the dropdown and note that there are 3 available options: *All*, *Metrics* and *Activity Log*. Leave the selection as **All** for now.

    > **Note**:  The signal types available for monitoring vary based on the selected target(s). The signal types may be metrics, log search queries or activity logs.

1. Under **Monitor service**, you will note that **All** is selected. Click on the dropdown and note that there are 3 available options: *All*, *Platform* and *Activity Log - Administrative*. Leave the selection as **All** for now.

    > **Note**:  The platform service provides metrics on service utiization, where as the activity log tracks administrative activities.

1. in the **Search by signal name** textbox, enter **connected** and this will immediately filter, then select **Connected devices (preview)** from the list below.

    The pane will update to display a chart similar to that you would create under **Metrics**, displaying the values associated with the selected signal (in this case *Connected devices (preview)*).

    Beneath the chart is the area that defines the **Alert logic**.

1. Under **Threshold** there are two possible selections - *Static* and *Dynamic*. You will notice that **Static** is selected and **Dynamic** is unavailable for this signal type.

    > **Note**:  As the names suggest, *Static Thresholds* specify a constant expression for the threshold, whereas *Dynamic Thresholds* detection leverages advanced machine learning (ML) to learn metrics' historical behavior, identify patterns and anomalies that indicate possible service issues. You can learn more about *Dynamic Thresholds* [here](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/alerts-dynamic-thresholds).

    We are going to create a static threshold that raises and alert whenever the *connected devices (preview)* signal is equal to 5 or more.

1. Under **Operator**, click the dropdown list and note the available operators. Select **Greater than or equal to**.

1. Under **Aggregation type**, click the dropdown and note the available options - select **Average**.

1. Under **Threshold value**, enter **5**.

    > **Note**:  The **Condition preview** refreshes to display the condition in an easier to read format.

    Below the **Condition preview** is the **Evaluation based on** area. The values herein determine the historical time period that is aggregated using the **Aggregation type** selected above and how often the condition is evaluated.

1. Under **Aggregation granularity (Period)**, select the dropdown and notice the available periods - select **5 minutes**.

1. Under **Frequency of evaluation**, select the dropdown and notice the available frequencies, select **Every 1 Minute**.

    > **Note**:  As the **Frequency of evaluation** is shorter than **Aggregation granularity (Period)**, this results in a sliding window evaluation. What this means is every minute, the preceding 5 minutes of values will be aggregated (in this case, averaged), and then evaluated against the condition. In a minutes time, again the preceding 5 minutes of data will be aggregated - this will include one minute of new data and four minutes of data that was already evaluated. Thus we have a sliding window that moves forward a minute at a time, but is always including 4 minutes of earlier data.

1. To configure the alert condition, click **Done**.

    The **Configure signal logic** pane closes and the **Create rule** pane appears. Notice that the **CONDITION** is now populated and a **Monthly cost in USD** is displayed. At the time of writing, the estimated cost of the alert condition is $0.10.

    Next, we need to configure the action taken when the alert condition is met.

1. Under **ACTIONS**, notice that no action group is selected. There are two options available - **Select action group** and **Create action group**. As we do not have an action group created yet, click **Create action group**.

    The **Add action group** pane is displayed.

    > **Note**:  An action group is a collection of notification preferences defined by the owner of an Azure subscription. An action group name must be unique within the Resource Group is is associated with. Azure Monitor and Service Health alerts use action groups to notify users that an alert has been triggered. Various alerts may use the same action group or different action groups depending on the user's requirements. You may configure up to 2,000 action groups in a subscription. You can learn more about creating and managing Action Groups [here](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/action-groups).

1. Next to **Action group name**, enter **AZ-220 Email Action Group**.

    > **Note**:  An action group name must be unique within the Resource Group is is associated with.

1. Next to **Short name**, enter **AZ220EmailAG**.

    > **Note**:  The short name is used in place of a full action group name when notifications are sent using this group and is limited to a max of 12 characters.

1. Next to **Subscription**, select the subscription you have been using for this lab.

1. Next to **Resource group**, select the resource group you are using for this lab - "AZ-220-RG".

    > **Note**:  Action Groups are usually shared across a subscription and would likely be centrally managed by the Azure subscription owner. As such they are more likely to be included in a common resource group rather than in a project specific resource group such as "AZ-220-RG". We are using "AZ-220-RG" to make it easier to clean up the resources after the lab.

1. In the next area, **Actions**, you can define a list of actions that will be performed whenever this action group is invoked.

1. Under **Action name**, enter **AZ220Notifications**.

1. Under **Action Type**, click the dropdown and notice the available options - select **Email/SMS/Push/Voice**.

    Immediately, the **Email/SMS/Push/Voice** action details pane is displayed. Notice that you can choose up to 4 methods for delivering the notification. For the purpose of this lab, we'll use **Email** and **SMS**.

1. Check **Email** and enter an email you wish to use to receive the alert.

1. Check **SMS**, enter your **Country code** and the **Phone number** you wish to receive the SMS alert.

1. Skip **Azure app Push Notifications** and **Voice**.

1. Finally, there is the option to **Enable the common alert schema** - select **Yes**.

   > **Note**:  There are many benefits to using the Common Alert Schema. It standardizes the consumption experience for alert notifications in Azure today. Historically, the three alert types in Azure today (metric, log, and activity log) have had their own email templates, webhook schemas, etc. With the common alert schema, you can now receive alert notifications with a consistent schema. You can learn more about the Common ALert6 Schema [here](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/alerts-common-schema).
   > **Important:** Given the benefits, you may wonder why the common alert schema is not enabled by default - well, when you select **Yes** you will see a warning **Enabling the common alert schema might break any existing integrations.** Bear this in mind in your own environments.

1. To save the **Email/SMS/Push/Voice** action configuration, click **OK**.

    The **Email/SMS/Push/Voice** pane closes and the list of **Actions** on the **Add action group** pane is updated. Notice that the new action has a link to **Edit details** if changes are required.

    At this point, we could add multiple actions if we needed to launch some business integration via *WebHooks* or an *Azure Function*, however for this lab, this notification is enough.

1. To create this action group, click **OK**.

    A few things happen at the same time. First, **Add action group** pane closes and the **Create rule** pane is displayed, with the new Action Group added to the list of **ACTIONS**.

    Then, in quick succession, you should receive both an SMS notification and an email, both of which inform you that you have been added to the **AZ220EmailAG** action group. In the SMS message, you will note that you can reply to the message to stop receiving future notifications and so on - you can learn more about the options [here](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/alerts-sms-behavior). In the email, you have links that you can click to view the details on action groups and, towards the bottom of the email (in a smaller font) you can see the option to unsubscribe.

1. Next, we configure the **ALERT DETAILS**.

1. Under **Alert rule name**, enter **Connected Devices Greater or Equal To 5**. The name should be descriptive enough to identify the alert.

1. Under **Description** you can optionally enter a more detailed description, enter **This alert is raised when the AZ-220-HUB  device connection threshold is greater than or equal to 5.**.

1. Under **Severity**, select the severity of the alert. In our scenario, this alert is *informational* and not indicative of any critical failure, therefore select **Sev 3**.

    > **Note**:  Severity options and the associated severity:
    >* Sev 0 = Critical
    >* Sev 1 = Error
    >* Sev 2 = Warning
    >* Sev 3 = Informational
    >* Sev 4 = Verbose

1. Under **Enable rule upon creation**, ensure **Yes** is selected.

    > **Note**:  It can take up to 10 minutes for a metric alert rule to become active.

1. To finally create the rule, click **Create alert rule**.

    The **Create rule** pane is closed and the list of **Alerts** is displayed. The **New alert rule** button that was previously displayed has now been replaced by **Manage alert rules(1)**. As we have yet to trigger any alerts, no alerts are listed here.

Now that we have create our alert, we should configure the environment we need for the device siumulation we will use to trigger the alert.

### Exercise 4: Simulating the Sensors

As part of the asset-tracking scenario, we need to have devices that simulate the tags that will be used to track the assets during transportation. As each device is activated, it should use automatic device provisioning to connect to the Iot solution and start sending telemetry. In order to automatically connect, each device will need its own X.509 certificate that is part of a chain to the root certificate used to create a group enrollment.

In this exercise, you will verify the existing environment, perform any necessary setup, generate 10 device certificates, and configure a console application that will simulate the 10 devices.

> **Note**: In lab 6 of this course (**Lab 6-Automatic Enrollment of Devices in DPS**) you configured DPS to use X.509 resources. If you still have that configuration available, that will shortcut a few steps in the tasks below. If you did not complete lab 6, make sure you complete every step below.

#### Task 1: Verify DPS Configuration

1. In your browser, navigate to the [Azure Portal](https://portal.azure.com/) and login to your subscription.

1. Navigate to the "AZ-220-RG" resource group and look for a **Device Provisioning Service** named **AZ-220-DPS-{YOUR INITIALS and DATE}**.

    > **Note**:  If the DPS does not exist, return to the **Setup Resources** task earlier in this lab.

1. To review the configuration of the DPS service, click  **AZ-220-DPS-{YOUR INITIALS and DATE}**.

1. To verify the certificate configuration, in the left hand navigation area, under **Settings**, click **Certificates**.

    > **Note**:  If the certificates list is empty, jump to the **Verify OpenSSL** section

1. Examine the certificate entry and ensure the **Status** for the certificate in the **Certificates** pane is displayed as **Verified**.

    > **Note**:  If the certificate status is **Unverified**, click the certificate to view the details, then click **Delete**. Enter the **Certificate Name** to confirm the deletion and click **OK**. Jump to the **Verify OpenSSL** section.

1. To verify the group enrollment, in the left navigation area, under **Settings** select **Manage enrollments**.

1. In the **Manage enrollments** pane, click on the **Enrollment Groups** link to view the list of enrollment groups in DPS.

    > **Note**:  Does the **simulated-devices** enrollment group **exist**? If so jump to the **Generate Device Certificates** in the next task.

1. If the  **simulated-devices** enrollment group does not exist, continue with the **Verify OpenSSL** section below.

#### Task 2: Verify OpenSSL

In the following steps you will verify that OpenSSL tools installed in an earlier lab are still available.

1. In your browser, navigate to the [Azure Shell](https://shell.azure.com/) and login to your subscription.

1. At the shell prompt, enter the following command:

    ```bash
    cd ~/certificates
    ```

    If you see an error that states **No such file or directory** then jump down to the **Install OpenSSL Tools** section below.

1. At the shell prompt, enter the following command:

    ```bash
    cd ~/certs
    ```

    If you see an error that states **No such file or directory** then jump down to the **Generate and Configure x.509 CA Certificates using OpenSSL** section below.

1. Jump down to the **Generate Device Certificates** in the next task.

#### Task 3: Install OpenSSL Tools

1. In the cloud shell, enter the following commands:

    ```bash
    mkdir ~/certificates

    # navigate to certificates directory
    cd ~/certificates

    # download helper script files
    curl https://raw.githubusercontent.com/Azure/azure-iot-sdk-c/master/tools/CACertificates/certGen.sh --output certGen.sh
    curl https://raw.githubusercontent.com/Azure/azure-iot-sdk-c/master/tools/CACertificates/openssl_device_intermediate_ca.cnf --output openssl_device_intermediate_ca.cnf
    curl https://raw.githubusercontent.com/Azure/azure-iot-sdk-c/master/tools/CACertificates/openssl_root_ca.cnf --output openssl_root_ca.cnf

    # update script permissions so user can read, write, and execute it
    chmod 700 certGen.sh
    ```

1. Continue to the **Generate and Configure x.509 CA Certificates using OpenSSL** section below.

### Task 4: Generate and Configure x.509 CA Certificates using OpenSSL

The first x.509 certificates needed are CA and intermediate certificates. These can be generated using the `certGen.sh` helper script by passing the `create_root_and_intermediate` option.

1. In the cloud shell, run the following command within the `~/certificates` directory of the **Azure Cloud Shell** to generate the CA and intermediate certificates:

    ```sh
    ./certGen.sh create_root_and_intermediate
    ```

1. The previous command generated a CA Root Certificate named `azure-iot-test-only.root.ca.cert.pem` is located within the `./certs` directory.

    Run the following command within the **Azure Cloud Shell** to download this certificate to your local machine so it can be uploaded to DPS.

    ```sh
    download ~/certificates/certs/azure-iot-test-only.root.ca.cert.pem
    ```

1. Navigate to the **Device Provisioning Service** (DPS) named `AZ-220-DPS-{YOUR-INITIALS-AND-CURRENT-DATE}` within the Azure portal.

1. On the **Device Provisioning Service** blade, click the **Certificates** link under the **Settings** section.

1. On the **Certificates** pane, click the **Add** button at the top to start process of uploading the x.509 CA Certificate to the DPS service.

    > **Note**:  If see an existing certificate, select and delete it.

1. On the **Add Certificate** pane, select the x.509 CA Certificate file in the **Certificate .pem or .cer file** upload field. This is the `azure-iot-test-only.root.ca.cert.pem` CA Certificate that was just downloaded.

1. Enter a logical name for the _Root CA Certificate_ into the **Certificate Name** field. For example, `root-ca-cert`

    This name could be the same as the name of the certificate file, or something different. This is a logical name that has no correlation to the _Common Name_ within the x.509 CA Certificate.

1. Click **Save**.

1. Once the x.509 CA Certificate has been uploaded, the **Certificates** pane will display the certificate with the **Status** of **Unverified**. Before this CA Certificate can be used to authenticate devices to DPS, you will need to verify **Proof of Possession** of the certificate.

1. To start the process of verifying **Proof of Possession** of the certificate, click on the **CA Certificate** that was just uploaded to open the **Certificate Details** pane for it.

1. On the **Certificate Details** pane, click on the **Generate Verification Code** button.

1. Copy the newly generated **Verification Code** that is displayed above the _Generate_ button.

    > **Note**:  You will need to leave the **Certificate Details** pane **Open** while you generate the Verification Certificate. If you close the pane, you will invalidate the Verification Code and will need to generate a new one.

1. Open the **Azure Cloud Shell**, if it's not still open from earlier, and navigate to the `~/certificates` directory.

1. **Proof of Possession** of the CA Certificate is provided to DPS by uploading a certificate generated from the CA Certificate with the **Validate Code** that was just generated within DPS. This is how you provide proof that you actually own the CA Certificate.

    Run the following command, passing in the **Verification Code**, to create the **Verification Certificate**:

    ```sh
    ./certGen.sh create_verification_certificate <verification-code>
    ```

    Be sure to replace the `<verification-code>` placeholder with the **Verification Code** generated by the Azure portal.

    For example, the command run will look similar to the following:

    ```sh
    ./certGen.sh create_verification_certificate 49C900C30C78D916C46AE9D9C124E9CFFD5FCE124696FAEA
    ```

1. The previous command generated a **Verification Certificate** that is chained to the CA Certificate with the Verification Code. The generated Verification Certificate named `verification-code.cert.pem` is located within the `./certs` directory of the Azure Cloud Shell.

    Run the following command within the **Azure Cloud Shell** to download this **Verification Certificate** to your local machine so it can be uploaded to DPS.

    ```sh
    download ~/certificates/certs/verification-code.cert.pem
    ```

1. Go back to the **Certificate Details** pane for the **CA Certificate** within DPS.

1. Select the newly created, and downloaded, **Verification Certificate** file, named `verification-code.cert.pem`, within the **Verification Certificate .pem or .cer file** field.

1. Click **Verify**.

1. With the **Proof of Possession** completed for the CA Certificate, notice the **Status** for the certificate in the **Certificates** pane is now displayed as **Verified**.

1. On the Device Provisioning Service settings pane on the left side, click **Manage enrollments**.

1. At the top of the blade, click **Add enrollment group**.

1. On the **Add Enrollment Group** blade, enter "**simulated-devices**" in the **Group name** field for the name of the enrollment group.

1. Ensure that the **Attestation Type** is set to **Certificate**.

1. Set the **Certificate Type** field to **CA Certificate**.

1. In the **Primary Certificate** dropdown, select the **CA Certificate** that was uploaded to DPS previously.

1. Notice the **Select the IoT hubs this group can be assigned to** dropdown has the **AZ-220-HUB-_{YOUR-ID}_** IoT Hub selected. This will ensure when the device is provisioned, it gets added to this IoT Hub.

1. In the Initial Device Twin State field, modify the `properties.desired` JSON object to include a property named `telemetryDelay` with the value of `"1"`. This will be used by the Device to set the time delay for reading sensor telemetry and sending events to IoT Hub.

    The final JSON will be like the following:

    ```js
    {
        "tags": {},
        "properties": {
            "desired": {
                "telemetryDelay": "1"
            }
        }
    }
    ```

1. Click **Save**

Now that the environment is setup, it's time to generate our device certificates.

### Exercise 5: Simulate Devices

In this task we will be generating X509 certificates from the root certifcate. We will then use these certificates in a console application that will simulate 10 devices connecting to DPS and sending telemetry to an IoT Hub.

#### Task 1: Generate Device Certificates

We will now generate and download 10 device certificates.

1. Open the **Azure Cloud Shell**, if it's not still open from earlier, and navigate to the `~/monitoring` directory.

1. To create an empty file in which we will copy the device generation script, enter the following commands:

    ```bash
    touch gen-dev-certs.sh
    chmod +x gen-dev-certs.sh
    ```

1. To edit the contents of the **gen-dev-certs.sh** file, use the **{ }** icon in Azure Cloud Shell to open the **Cloud Editor**.

    To open the **gen-dev-certs.sh** file, you will have to expand the **monitoring** node in the **Files** list to locate it.

1. Paste the following code into the cloud editor:

    ```bash
    #!/bin/bash

    # Generate 10 device certificates
    # Rename for each device
    # download from the Cloud CLI
    pushd ~/certificates
    for i in {1..10}
    do
        chmod +w ./certs/new-device.cert.pem
        ./certGen.sh create_device_certificate asset-track$i
        sleep 5
        cp ./certs/new-device.cert.pfx ./certs/new-asset-track$i.cert.pfx
        download ./certs/new-asset-track$i.cert.pfx
    done
    popd
    ```

    This script will create and download 10 device certificates.

1. To save the edited **gen-dev-certs.sh** file, press **CTRL-Q**. If prompted to save you changes before closing the editor, click **Save**.

1. To run the **gen-dev-certs.sh** script, run the following:

    ```bash
    ./gen-dev-certs.sh
    ```

    While the script runs, you will see the output from the certificate generator and then the browser should automatically download each certificate in turn. Once it completes, you will have 10 certificates available in your browser download location:

    * new-asset-track1.cert.pfx
    * new-asset-track2.cert.pfx
    * new-asset-track3.cert.pfx
    * new-asset-track4.cert.pfx
    * new-asset-track5.cert.pfx
    * new-asset-track6.cert.pfx
    * new-asset-track7.cert.pfx
    * new-asset-track8.cert.pfx
    * new-asset-track9.cert.pfx
    * new-asset-track10.cert.pfx

With these certificates available, you are ready to configure the device simulator.

#### Task 2: Add Certificates to Simulator

1. Copy the downloaded **x.509 Device Certificate** files to the `/LabFiles` directory; within the root directory along-side the `Program.cs` file. The **Simulated Devices** project will need to access this certificate file when authenticating to the Device Provisioning Service.

    After copied, the certificate files will be located in the following locations:

    ```text
    /LabFiles/new-asset-track1.cert.pfx
    /LabFiles/new-asset-track2.cert.pfx
    /LabFiles/new-asset-track3.cert.pfx
    /LabFiles/new-asset-track4.cert.pfx
    /LabFiles/new-asset-track5.cert.pfx
    /LabFiles/new-asset-track6.cert.pfx
    /LabFiles/new-asset-track7.cert.pfx
    /LabFiles/new-asset-track8.cert.pfx
    /LabFiles/new-asset-track9.cert.pfx
    /LabFiles/new-asset-track10.cert.pfx
    ```

1. Using **Visual Studio Code**, open the `/LabFiles` folder.

1. Open the `Program.cs` file.

1. Locate the `GlobalDeviceEndpoint` variable, and notice it's value is set to `global.azure-devices-provisioning.net`. This is the **Global Device Endpoint** for the Azure Device Provisioning Service (DPS) within the Public Azure Cloud. All devices connecting to Azure DPS will be configured with this Global Device Endpoint DNS name.

    ```csharp
    private const string GlobalDeviceEndpoint = "global.azure-devices-provisioning.net";
    ```

1. Locate the `dpsIdScope` variable, and replace the value with the **ID Scope** of the Device Provisioning Service.

   ```csharp
   private static string dpsIdScope = "<DPS-ID-Scope>";
   ```

   We need to replace the `<DPS-ID-Scope>` value with the actual value.

1. Return to the Azure cloud shell and enter the following command:

    ```bash
    az iot dps show --name AZ-220-DPS-{YOUR-INITIALS-AND-CURRENT-DATE} --query properties.idScope
    ```

    > **Note**:  Ensure you use the name of your DPS instance above.

    Copy the output of the command and replace the `<DPS-ID-Scope>` value in Visual Studio code. It should look similar to:

   ```csharp
   private static string dpsIdScope = "0ne000A6D9B";
   ```

This app is very similar to the app used in the earlier lab **L06-Automatic Enrollment of Devices in DPS**. The primary difference is that instead of just enrolling a single device simulator and then sending telemetry, it instead enrolls 10 devices, one every 30 seconds. Each simulated device will then send telemetry. This should then cause our alert to be raised and log monitoring data to storage.

#### Task 3: Run the Simulator

1. To run the app, in Visual Studio Code, open a terminal, and enter the following command:

    ```bash
    dotnet run
    ```

    You should see output that shows the first device being connected via DPS and then telemetry being sent. Every 30 seconds thereafter, and additional device will be connected and commence sending telemetry until all 10 devices are connected and sending telemetry.

1. Return to the DPS group enrollment in the Azure Portal.

1. In the **simulated-devices** enrollment group, to view the connected devices, click **Registration Records**.

    You should see a list of the devices that have connected. You can hit **Refresh** to update the list.

    Now that we have the devices connected and sending telemetry, we await the triggering of the alert once we have 5 or more devices connected for 5 mins. You should receive and SMS message that looks similar to:

    ```text
    AZ220EmailAG:Fired:Sev3 Azure Monitor Alert Connected Devices Greater or Equal to 5 on <your IoT Hub>
    ```

    The email will look similar to:

    ![Email Alert](./Media/M99-L17-04-email-alert.png)

1. Once the alerts have arrived, you can exit the application by either hitting **CTRL+C** in the Visual Studio Code terminal, or by closing Visual Studio Code.

    > **Note**:  When the devices are disconnected, you will receive messages informing you the alert has been resolved.

Now, let's check the storage account to see if anything has been logged by Azure Monitor.

### Exercise 6: Review Metrics, Alerts and Archive

In this exercise, you will examine some of the reporting and logging resources that you configured earlier in this lab, and review whatever event data has been recorded in the short time that has elapsed.  

#### Task 1: See the Metrics in the Portal

1. In the Azure portal, open the Metrics chart that you pinned to the dashboard by clicking on the chart title.

    The chart will open and fill the page.

1. Change the time values to the **Last 30 minutes**.

    Notice that you can see *Telemetry messages sent* and *Connected devices (preview)** values, with the most recent numbers at the bottom of the chart - move you mouse over the chart to see values a specific points in time.

    ![metrics chart](./Media/M99-L17-05-metrics-chart.png)

#### Task 2: See the Alerts

To use the Azure Portal to review alerts, complete the following steps.

1. In the Azure Portal, in the search box at the top of the screen, enter **Monitor** and the select **Monitor** from the list, under **Service**.

    The **Monitor - Overview** page is displayed. This is the overview for all of the monitoring activities for the current subscription.

1. In the left hand navigation, select **Alerts**.

    This alerts view shows all alerts for all subscriptions. Let's filter this to the IoT Hub.

1. At the top of the page, under **Subscription**, select the subscription you are using.

1. Under **Resource group**, select "AZ-220-RG".

1. Under **Resource**, select **AZ-220-HUB-{YOUR-INITIALS-AND-CURRENT-DATE}**.

1. Under **Time range**, select **Past hour**.

    You should now see a summary of alerts for the last hour. Under **Total alert rules** you should see **1**, the alert you created earlier. Below this, you will see a list of the severity categories as well as the count of alerts per category. The alerts we are interested in are **Sev 3**. You should see at least one (if you have stopped and restarted the device simulator, you may have generated more that one alert).

1. In the list of severities, click **Sev 3**.

    the **All Alerts** page will open. At the top of the page you will see a number of filter fields - these have been populated with the values from the preceding screen so that only the **Sev 3** alerts for the selected IoT hub are shown. It will show you the alerts that are active, and if there are any warnings.

1. Select an alert from the list.

    A pane will open showing a **Summary** of the details for the alert. This includes a chart illustrating why the alert fired - a dash line shows the threshold value as well as the current values for the monitored metric. Below this are details of the **Criterion** and other details.

1. At the top of the pane, below the title, click **History**.

    In this view you can see when the alert fired, the action group that was invoked, and any other changes such as when the alert is resolved and so on.

1. At the top of the pane, below the title, click **Diagnostics**.

    If there were any issues related to the alert, addition details would be shown here.

#### Task 3: See the Diagnostic Logs

Earlier, you set up your diagnostic logs to be exported to blob storage. Let's check to see what was written.

1. Navigate to your "AZ-220-RG" resource group.

1. In the list of resources, select the Storage Account that was created earlier - **az220storage{YOUR-INITIALS-AND-CURRENT-DATE}**.

    The **Overview** for the storage account will be displayed.

1. Scroll down until you can see the metrics charts for the Storage Account: *Total egress*, *Toral ingress*, *Average latency* and *Request breakdown*.

    You should see that there is activity displayed.

1. To view the data that has been logged, in the left hand navigation area, select **Storage explorer (preview)**.

1. In the **Storage explorer** pane, expand the **BLOB CONTAINERS** node.

    When Azure Monitor first sends data to a storage account, it creates a container called **insights-logs-connection**.

1. Select the **insights-logs-connection** container - the contents of the container will be listed to the right.

    Logs are written to the container in a very nested fashion. You will need to open each subfolder in turn to navigate to the actual log data. The structure is similar to that show below:

    * **resourceId=**
      * **SUBSCRIPTIONS**
        * **<GUID>** - this is the ID for the subscription that generated the log
          * **RESOURCEGROUPS** - contains a folder for each resource group that generated a log
            * "AZ-220-RG" - the resource group that contains the IoT Hub
              * **PROVIDERS**
                * **MICROSOFT.DEVICES**
                  * **IOTHUBS**
                    * **AZ-220-HUB-{YOUR-INITIALS-AND-CURRENT-DATE}** - contains a folder for each year where a log was generated
                      * **Y=2019** - contains a folder for each month where a log was generated
                        * **m=12** - contains a folder for each day where a log was generated
                          * **d=15** - contains a folder for each hour where a log was generated
                            * **h=15** - contains a folder for each minute where a log was generated
                              * **m=00** - contains the log file for that minute

    Drill down until you get to the current date and select the most recent file.

1. With the file selected, in the toolbar at the top of the pane, click **Download**.

1. Open the downloaded file in Visual Studio Code.

    You should see a number of lines of JSON.

1. To make the JSON easier to read, press **F1**, enter **Format document** and select **Format document** from the list of options.

    The JSON will show a list of connection and disconnection events similar to:

    ```json
    {
        "time": "2019-12-26T14:32:45Z",
        "resourceId": "/SUBSCRIPTIONS/AE82FF3B-4BD0-462B-8449-D713DD18E11E/RESOURCEGROUPS/AZ-220/PROVIDERS/MICROSOFT.DEVICES/IOTHUBS/AZ-220-HUB-DM121619",
        "operationName": "deviceConnect",
        "category": "Connections",
        "level": "Information",
        "properties": "{\"deviceId\":\"asset-track9\",\"protocol\":\"Amqp\",\"authType\":\"{\\\"scope\\\":\\\"device\\\",\\\"type\\\":\\\"x509Certificate\\\",\\\"issuer\\\":\\\"external\\\",\\\"acceptingIpFilterRule\\\":null}\",\"maskedIpAddress\":\"67.176.115.XXX\",\"statusCode\":null}",
        "location": "westus"
    }
    {
        "time": "2019-12-26T14:33:12Z",
        "resourceId": "/SUBSCRIPTIONS/AE82FF3B-4BD0-462B-8449-D713DD18E11E/RESOURCEGROUPS/AZ-220/PROVIDERS/MICROSOFT.DEVICES/IOTHUBS/AZ-220-HUB-DM121619",
        "operationName": "deviceConnect",
        "category": "Connections",
        "level": "Information",
        "properties": "{\"deviceId\":\"asset-track10\",\"protocol\":\"Amqp\",\"authType\":\"{\\\"scope\\\":\\\"device\\\",\\\"type\\\":\\\"x509Certificate\\\",\\\"issuer\\\":\\\"external\\\",\\\"acceptingIpFilterRule\\\":null}\",\"maskedIpAddress\":\"67.176.115.XXX\",\"statusCode\":null}",
        "location": "westus"
    }
    {
        "time": "2019-12-26T14:37:29Z",
        "resourceId": "/SUBSCRIPTIONS/AE82FF3B-4BD0-462B-8449-D713DD18E11E/RESOURCEGROUPS/AZ-220/PROVIDERS/MICROSOFT.DEVICES/IOTHUBS/AZ-220-HUB-DM121619",
        "operationName": "deviceDisconnect",
        "category": "Connections",
        "level": "Information",
        "properties": "{\"deviceId\":\"asset-track8\",\"protocol\":\"Amqp\",\"authType\":null,\"maskedIpAddress\":\"67.176.115.XXX\",\"statusCode\":null}",
        "location": "westus"
    }
    {
        "time": "2019-12-26T14:37:29Z",
        "resourceId": "/SUBSCRIPTIONS/AE82FF3B-4BD0-462B-8449-D713DD18E11E/RESOURCEGROUPS/AZ-220/PROVIDERS/MICROSOFT.DEVICES/IOTHUBS/AZ-220-HUB-DM121619",
        "operationName": "deviceDisconnect",
        "category": "Connections",
        "level": "Information",
        "properties": "{\"deviceId\":\"asset-track4\",\"protocol\":\"Amqp\",\"authType\":null,\"maskedIpAddress\":\"67.176.115.XXX\",\"statusCode\":null}",
        "location": "westus"
    }
    ```

    Notice that each individual entry is a single JSON record - the overall document is not a a valid JSON document. Within each record you can see details relating to the originating IoT Hub and **properties** for each event. Within the **properties** object, you can see the connecting (or disconnecting) **deviceId**.
