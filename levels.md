# Logger Workshop: Levels

When instrumenting your code (i.e. adding Logger statements to it) you should not have to add and remove statements while testing. You only need to add the statement once then you can configure whether or not you want to actually store the logged message via a configuration.

This is where logging levels comes in. When calling Logger you explicitly tell it which logging level to use. The API that is used and the level that is configured will determine if the message is actually stored in the table. If the level is not configured to be written Logger immediately exits the call to help reduce performance impacts.

More information about levels is available in the [Best Practices](https://github.com/OraOpenSource/Logger/blob/master/docs/Best%20Practices.md#logger-level-guide) doc.

## Levels

Run the following script which will highlight how levels work and the different types of levels

```sql
begin
  logger.log('This is a debug message. (level = DEBUG)');
  logger.log_information('This is an informational message. (level = INFORMATION)');
  logger.log_warning('This is a warning message. (level = WARNING)');
  logger.log_error('This is an error message (level = ERROR)');
  logger.log_permanent('This is a permanent message, good for upgrades and milestones. (level = PERMANENT)');
end;
/
```

You can then see the differences in the `logger_level` column:

```sql
select id, logger_level, text
from logger_logs_5_min
order by id desc;
```

## Changing Levels

To change a level call [`logger.set_level`](https://github.com/OraOpenSource/Logger/blob/master/docs/Logger%20API.md#set_level). Set the level to `error` mode. This is configuration is commonly used in production environments to help reduce the number of messages logged.

```sql
exec logger.set_level(logger.g_error);
```

Now run the same script from above. Notice only two rows are logged.

```sql
begin
  logger.log('This is a debug message. (level = DEBUG)');
  logger.log_information('This is an informational message. (level = INFORMATION)');
  logger.log_warning('This is a warning message. (level = WARNING)');
  logger.log_error('This is an error message (level = ERROR)');
  logger.log_permanent('This is a permanent message, good for upgrades and milestones. (level = PERMANENT)');
end;
/
```

```sql
select id, logger_level, text
from logger_logs_5_min
order by id desc;
```

Try setting different logging levels and run the above scripts to see the differences. The Logger [level global constants](https://github.com/OraOpenSource/Logger/blob/master/docs/Logger%20API.md#numeric) should be used for each setting.

## Cleanup

The rest of this workshop depends on the fact that the Logger level is set to `debug`. **Before continuing change the level to `debug`**

```sql
exec logger.set_level(logger.g_debug);
```
