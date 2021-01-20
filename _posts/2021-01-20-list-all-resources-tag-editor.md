---
title: How to List All AWS Resources within an Account Using AWS Tag Editor
tags:
- aws
- TagEditor
- Tags
---

As a DevOps or Cloud Engineer or even as a manager, there are times when you need to list all the resources in an AWS account. This could be for purposes like inventory management, cost control, or ensuring compliance with organizational policies.

Personally, I’ve often been asked by non-technical stakeholders/ clients to provide a comprehensive list of all resources in their AWS account. While CLI scripts can also achieve this, I found using AWS Tag Editor to be a quicker and more straightforward solution—especially when managing multiple accounts or when the request comes on short notice.

Here’s how you can use AWS Tag Editor for this purpose.

---
## Listing All AWS Resources

### Open the AWS Tag Editor

1. Go to the [AWS Management Console](https://aws.amazon.com/console/).
2. Look for the **Resource Groups** option in the top menu.
3. Click on **Tag Editor** from the dropdown menu.

![image info](assets/images/resource_group.png){: width="650" }
### Selecting Regions

- Choose the regions where your resources are located.
- You can pick one / multiple regions.

### Choosing Resource Types

- Select specific resource types (like EC2 instances, S3 buckets, etc.) or choose **All resource types** to view everything.

![image info](assets/images/Tag_Editor.png){: width="650" }

### Adding Tags (Optional)

- Add specific tags in the **Tags** section to filter resources, if needed.
- Leave this blank to list all resources without filtering.

### Searching for Resources

- Click on the **Search resources** button to list resources based on your selected criteria.

### Viewing and Managing Resources

- The list of resources will be displayed.
- Click on each resource to view details or manage tags.

### Exporting the List (Optional)

- Use the **Export to CSV** button to download the resource list.
- The CSV file will include details of all listed resources.

---

## So..

While AWS Tag Editor is primarily for tagging resources, it’s a handy tool to quickly list all your AWS resources. By following these steps, you can easily organize and manage your resources.

Remember, tagging not only helps in organizing resources but also in managing costs and maintaining security. Always use meaningful tags for better resource management.
