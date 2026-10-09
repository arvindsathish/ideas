A large enterprise is conducting a hackathon and hackathon will be on an Azure lab environment. Analyze the information, come up with 5 project ideas, highlevel plan to win the hackathon with a successful demo using the list of tools. Also include how the tools will work together for the given ideas

"Use case title: Business Role Health and Operational Efficiency

Business Problem:
Failure to maintain business roles results in outdated access packages that no longer align with employees responsibilities.
During employee transfers or role changes, users must submit numerous access requests to gain the access needed for their new positions.
This increases ticket volume, delays productivity and reduces the overall effectiveness of role-based access management

Additional context:
A business role is a governed collection of application access, entitlements and permissions that represents the standard access required for a specific job function.
Business roles are maintained using Sailpoint tool and are used to automate access provisioning support compliance requirements and enable RBAC across the enterprise.
Business roles are uniquely defined using three key attributes in the HR tool and drive the assigment of business roles.
Job Title
Cost center
Business Group (Country)

Whenever one or more of these attributes changes in the HR tool, an event is sent to Sailpoint tool amd Sailpoint adjusts access as part of the Joiner, Mover or Leaver process.
Employee's manager is the designated business role owner and responsible for ensuring the role accurately reflects the access required for employees performing their job functions and for maintaining the role as business processes, applications and access requirements evolve. Role owners need to certify the role assignments periodically to ensure alignment with security, compliance and operational requirements.

Business roles plays a critical part in Joiner, Mover or Leaver process. During a mover event, Sailpoint reevaluates access based on the employee's new job Title, Cost Center and Business Group.
If the business role is not properly maintained, the employee may lose access to systems necessary to perform their job responsibilities and cause productivity impact.
Most of the time, an employee ends up losing their access on some of the tools whenever there is a role change.

Libraries/Frameworks/Tools allowed for hackathon:
Azure AI Search
Azure Logic Apps
Azure SQL Database
Datafactory
Power BI
Entra ID
ServiceNow APIs (only Mock API)"

Hackathon admins have asked to use synthetic data to mimic HR tool and Sailpoint. As the Azure lab environment will not have access to the HR tool, ServiceNow and Sailpoint due to compliance and security reason. Even for ServiceNow either synthetic data or local mock API should be used.