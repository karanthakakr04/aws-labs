# S3 Multi-Region Access Points

## Overview

In this mini project, we're going to create an S3 Multi-Region Access Point (MRAP), which allows a single S3 endpoint to distribute traffic to the closest bucket to the user or application.

Multiple buckets are added to the Multi-Region Access Point in different regions. S3 requests (GETs, PUTs, etc.) are routed to the closest bucket to the user, minimizing latency. Optionally, changes can be replicated between all buckets to ensure data consistency across regions.

For this project, we'll be using two buckets: one in `ap-southeast-2` (Sydney) and one in `ca-central-1` (Canada Central).

## Setup Instructions

### Stage 1 - Create the buckets

1. Head to the S3 dashboard: [https://s3.console.aws.amazon.com/s3/buckets](https://s3.console.aws.amazon.com/s3/buckets)
2. Click on **Create bucket**
3. For the first bucket:
   - Set the **Bucket Name** to `multi-region-access-point-demo-sydney`
   - Set the **region** to `ap-southeast-2`
4. For the second bucket:
   - Set the **Bucket Name** to `multi-region-access-point-demo-canada`
   - Set the **region** to `ca-central-1`
5. Under **Bucket Versioning**, select **Enable** for both buckets.
6. Leave everything else as default and click **Create bucket**

### Stage 2 - Create Multi-Region Access Point

1. Head to the S3 dashboard: [https://s3.console.aws.amazon.com/s3/buckets](https://s3.console.aws.amazon.com/s3/buckets)
2. Go to the **Multi-Region Access Points** page, and click **Create Multi-Region Access Point**
   ![screenshot-1](https://github.com/karanthakakr04/aws-labs/assets/17943347/f3dda330-1fc8-4d94-8681-871d97d596d1)
   ![screenshot-2](https://github.com/karanthakakr04/aws-labs/assets/17943347/d2bc82ed-39bf-4747-b62d-24eb0e5418d0)
3. Set the **Multi-Region Access Point name** to anything you like, these are not globally unique.
4. Click **Add buckets**, select your two buckets, and click **Add buckets**.
   ![screenshot-3](https://github.com/karanthakakr04/aws-labs/assets/17943347/5ff5f862-f388-4946-85bd-7700317065c2)
   ![screenshot-4](https://github.com/karanthakakr04/aws-labs/assets/17943347/4cf11303-0475-498d-882e-a40c7ba9f8c1)
5. Leave everything else as default, and click **Create Multi-Region Access Point**
   - **Note:** This process can take "up to" 24 hours, but in my testing, it took <15 minutes.

### Stage 3 - Setting up bucket replication

1. Head to the S3 dashboard: [https://s3.console.aws.amazon.com/s3/buckets](https://s3.console.aws.amazon.com/s3/buckets)
2. Go to the **Multi-Region Access Points** page, click on the access point that you created earlier, and go to the **Replication and failover** page.
3. Under **Replication rules**, click on **Create replication rules**
   ![screenshot-5](https://github.com/karanthakakr04/aws-labs/assets/17943347/1dcf9af9-fbcb-42c3-a693-20671cadbb34)
4. Leave **Replicate objects among all specified buckets** selected, and check all of the buckets in this access point.
   ![screenshot-6](https://github.com/karanthakakr04/aws-labs/assets/17943347/636a9139-efb0-4ff6-a8d9-f421e0bcbd28)
5. Under **Scope** select **Apply to all objects in the bucket**
6. Leave everything else as is, and click **Create replication rules**
   - We now have both buckets replicating to each other.

### Stage 4 - Testing using the MRAP

1. Head to the S3 dashboard: [https://s3.console.aws.amazon.com/s3/mraps](https://s3.console.aws.amazon.com/s3/mraps)
2. Go to the **Multi-Region Access Points** page, select your access point, and click **Copy ARN**. We will need this for the next step.
   ![screenshot-7]()
4. We're going to use the AWS CloudShell in the console to test this out, as it allows us to connect to S3 from different regions. If you connected from your local PC, you would always be routed to the closest bucket to your ISP.
   - **Note:** CloudShell is not available in every region, see here for a list of available regions you can use: [https://docs.aws.amazon.com/general/latest/gr/cloudshell.html](https://docs.aws.amazon.com/general/latest/gr/cloudshell.html)
   - Be aware of the potential cost implications of using AWS CloudShell.
5. Make sure you're in the region you want to connect from, and open up CloudShell.
   ![screenshot-8]()
7. In the CloudShell console, we'll create a 10MB file named `test1.file`, and upload it to the S3 MRAP ARN you copied earlier.

   ```bash
   dd if=/dev/urandom of=test1.file bs=1M count=10
   aws s3 cp test1.file s3://<MRAP_ARN>/
   ```

   - Replace `<MRAP_ARN>` with the actual ARN of your Multi-Region Access Point.

8. Now, because I've done this from Tokyo, Sydney should be the closest bucket and should have the file. If we check the Canada bucket, it should already be replicated.
   ![screenshot-9]()
   ![screenshot-10]()
   - **Note:** S3 replication isn't guaranteed to complete in a set time. In fact, their documentation says it can take hours or longer. To get around this, you can enable Replication Time Control (RTC) which speeds up replication and advertises 99.99% of objects replicated within 15 minutes, and "most" objects replicated in seconds. This costs extra and isn't required for our demo.

10. Let's switch to another region in CloudShell, in my case, I'm going to us Ohio (us-east-2). Again, run these two commands, changing the file name to `test2.file`:

   ```bash
   dd if=/dev/urandom of=test2.file bs=1M count=10
   aws s3 cp test2.file s3://<MRAP_ARN>/
   ```

   - Replace `<MRAP_ARN>` with the actual ARN of your Multi-Region Access Point.
     ![screenshot-11]()
     ![screenshot-12]()

11. As another test, I'm going to pick a region as close to the center of both buckets as I can, and see which bucket receives the file first. In my case, this is Mumbai (ap-south-1).
   - **Note:** While Mumbai may geographically be close to the center, there are a lot of network factors behind the scenes which control which region is closest.
     ![screenshot-13]()

   ```bash
   dd if=/dev/urandom of=test3.file bs=1M count=10
   aws s3 cp test3.file s3://<MRAP_ARN>/
   ```

   - Replace `<MRAP_ARN>` with the actual ARN of your Multi-Region Access Point.
     ![screenshot-14]()
     ![screenshot-15]()

11. As a final test, we'll see what happens if we try to get an object, via our Multi-Region Access Point, that has been created in one bucket, but our 'get' request is routed to another bucket that has not had the file replicated yet.
   - To do this, you will need to have two CloudShell's open, one nearest to each bucket.
   - In one window, create a new file and upload it:

     ```bash
     dd if=/dev/urandom of=test4.file bs=1M count=10
     aws s3 cp test4.file s3://<MRAP_ARN>/
     ```

   - In another window, try to copy that file to your instance:

     ```bash
     aws s3 cp s3://<MRAP_ARN>/test4.file .
     ```

   - If you're not quick enough, or replication is unusually fast, you might miss out, and should start again (with a new file name).
   - **Note:** If your application requires all objects to be available immediately, Multi-Region Access Points may not be the best solution, or you should at least ensure your application can handle 404 errors.

## Cleanup

By following these cleanup steps, you will remove the Multi-Region Access Point and delete the associated S3 buckets, effectively cleaning up the resources used in the setup.

1. Delete the Multi-Region Access Point:
   - Go to the S3 dashboard: [https://s3.console.aws.amazon.com/s3/mraps](https://s3.console.aws.amazon.com/s3/mraps)
   - Navigate to the **Multi-Region Access Points** page.
   - Select the access point you created earlier.
   - Click on the **Delete** button to delete the access point.
   - **Note:** The deletion process may take a few minutes, and you might not be able to delete the associated buckets while the access point deletion is in progress.
     ![screenshot-16]()

2. Empty the S3 buckets:
   - Go to the S3 dashboard: [https://s3.console.aws.amazon.com/s3/buckets](https://s3.console.aws.amazon.com/s3/buckets)
   - Navigate to the **Buckets** page.
   - For each bucket you created (`multi-region-demo-sydney` and `multi-region-demo-canada`):
     - Select the bucket.
     - Click on the **Empty** button.
     - In the confirmation dialog, enter `permanently delete` to confirm the action.
     - Click on the **Empty** button to empty the bucket.
       ![screenshot-17]()

3. Delete the S3 buckets:
   - Once both buckets are empty, go back to the **Buckets** page.
   - For each bucket (`multi-region-demo-sydney` and `multi-region-demo-canada`):
     - Select the bucket.
     - Click on the **Delete** button.
     - In the confirmation dialog, enter the bucket name to confirm the deletion.
     - Click on the **Delete** button to delete the bucket.
   - **Note:** Ensure that the bucket emptying process is complete before attempting to delete the buckets.
     ![screenshot-18]()

## Conclusion

In this readme, we explored the process of setting up and testing Amazon S3 Multi-Region Access Points. By creating a global endpoint and configuring replication between buckets, we were able to achieve improved performance and reduced latency when accessing data across multiple regions.

S3 Multi-Region Access Points provide a powerful and flexible solution for managing data in a multi-region environment. However, it's important to consider factors such as replication time and application requirements when deciding if MRAPs are the best fit for your use case.

Feel free to explore further configurations and experiment with different scenarios based on your specific needs. Happy testing!

## References

- [Amazon S3 Multi-Region Access Points](https://aws.amazon.com/s3/features/multi-region-access-points/)
- [Getting started with Amazon S3 Multi-Region Access Points](https://aws.amazon.com/getting-started/hands-on/getting-started-with-amazon-s3-multi-region-access-points/)
