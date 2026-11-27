# Part 0: Sign in to Azure and review your resources

In this exercise, you will sign in to the Azure portal, review your lab resources, and confirm the HorizonDB parameter group configuration.

## Task 1: Sign in to Azure and review lab resources

In this section, we'll open Microsoft Edge in the lab environment and sign in to the Azure portal to review the resources we'll use in this lab.

1. Double-click on the **Microsoft Edge** icon on the desktop.

    ![Starting the Lab](../img/0-portal.png)

> [!NOTE]
> If you see a sign-in pop-up, select **Not now**.

1. To sign in to the Azure portal, enter the following credentials:

    - Username: +++@lab.CloudPortalCredential(User1).Username+++
    - TAP: +++@lab.CloudPortalCredential(User1).TAP+++

> [!NOTE]
> This lab uses a Temporary Access Pass (TAP) instead of a password to sign in to Azure. If you need to access this TAP code again, use the **Resources** tab at the top of these lab instructions.

1. Once you've signed in to the Azure portal, select **View all resources**.

    ![View all resources](../img/0.1-view-all-resources.png)

1. We'll be using these resources during this lab:
    - A Microsoft Foundry instance
    - An Azure HorizonDB database instance

    ![Azure Lab Resources](../img/0.2-lab-resources.png)

## Task 2: Verify the HorizonDB parameter group

Your HorizonDB resource has been provisioned along with a **parameter group** containing the server parameter configuration for your PostgreSQL extensions and the associated allowlist details. The parameter group is associated with the cluster's compute pool.

Parameter groups are a new concept in HorizonDB. They act as containers for PostgreSQL configuration values that can be applied to one or more compute pools. Instead of managing configuration settings for each pool individually, you can define them in a parameter group and associate that group with multiple pools to ensure consistency across your environment. This lab has only one cluster and one pool, but we still need to confirm that the parameter group has been enabled.

Parameter groups are first-class resources in Azure and are surfaced within the specific resource group and subscription defined in their resource identifier.

1. Select the HorizonDB instance in the portal.

1. Expand **Settings** in the left navigation and select **Parameters**.

1. Confirm that your parameter group has been enabled for your HorizonDB instance.

    ![HorizonDB Server Parameters](../img/0.3-server-parameters.png)

## Next Steps

> Select **Next >** to proceed to the next part of the lab.
