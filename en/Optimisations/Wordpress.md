# Optimisations for Wordpress cluster


## Error with *zero dates*

According to the configuration of MySQl and the Wordpress version, it is possible that you cannot create new post on the admin side because the database does not accept dates where all fields are 0, it is what Wordpress provides temporarily.

To solved this issue, it is necessary to delete two variables from MySQL "nodes". **On each MYSQL nodes** of you environment, follow these actions below : 

1.Connect to PHPMyAdmin with the credentiales sent by e-mail
2.On the dashboard, navigate to *Variables* tab of your MYSQL nodes
3.Search the vairable `sql_mode` and delete `NO_ZERO_IN_DATE,NO_ZERO_DATE` in the value
4. Save your the new configuration

You can now add posts in Wordpress admin.

>To have more information regarding the issue, you can check here : https://de-ch.wordpress.org/plugins/incorrect-datetime-bug-plugin-fix/
