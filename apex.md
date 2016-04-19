# Logger Workshop: APEX

Logger has some built in features specific for APEX developers. In this section you'll leverage these tools to enhance logging for your APEX applications.

## Setup

An application will reside in your assigned workspace. Edit and run the application.

## Log CGI Variables

CGI variables contain a lot of connection and browser information about the user. In certain situations it can be very helpful to quickly view this information. [`logger.log_cgi_env`](https://github.com/OraOpenSource/Logger/blob/master/docs/Logger%20API.md#log_cgi_env) will log all this information in one simple call. Please review the API for more information and parameters.

### Exercise

On P1 there is a Dynamic Action (DA) called `onClick Log CGI`.

- Edit `Execute PL/SQL Code` and add the following `logger.log_cgi_env;`
- Save the page
- Run P1 and click on `Log CGI Env` button
- In SQL Dev view the logs and see if you can see all the CGI variables.
  - _Hint: following the instructions in the `text` column_
  - To view all the CGI information that is logged you may want to copy the appropriate data and paste into your text editor.

## Log APEX Items

Sometimes you may want to log all the APEX items. [`logger.log_apex_items`](https://github.com/OraOpenSource/Logger/blob/master/docs/Logger%20API.md#log_apex_items) will easily store all the values for you. It has the option to store all, application, or page items. Please view the documentation for more info.

### Exercise

On P1 there is a Dynamic Action (DA) called `onClick Log APEX Item`.

- Edit `Execute PL/SQL Code` and add the following `logger.log_apex_items;`
- Save the page
- Run P1
- Edit one of the employees then click `Apply Changes` (_you don't need to make any changes_). The purpose of doing this is to register some content in Session State.
- On P1 click on the `Log APEX Items` button

APEX items are stored in a separate table. The following code will highlight how to find them.

```sql
select *
from logger_logs_5_min
order by 1 desc;

-- Take note of the ID of the APEX items

select *
from logger_logs_apex_items
where 1=1
  and log_id = :log_id;
```

You should now see all the APEX items.


## APEX Error Handling Function

APEX allows you to provide a custom error handling function. This is excitedly useful for various reasons, especially when you want to apply custom logging. This section will tie together the procedures that were covered on this page along with the rest of the workshop.


In SQL Dev, compile the following function. The skeleton of the function was taking from the [APEX Error Example](https://docs.oracle.com/cd/E59726_01/doc.50/e39149/apex_error.htm#AEAPI2216):

```sql
create or replace function apex_error_handling_example (
  p_error in apex_error.t_error )
  return apex_error.t_error_result
is
  l_result          apex_error.t_error_result;
  l_reference_id    number;
  l_constraint_name varchar2(255);
  l_reference varchar2 (255);

  l_scope logger_logs.scope%type := lower($$plsql_unit);
  l_params logger.tab_param;

  procedure p_unhandled_exception
  as
  begin
    l_reference := to_char(apex_application.g_flow_id)||to_char(systimestamp, '.YYYY.MM.DD.SSSSS');

    l_result.message := 'An error has occured. Reference: %REFERENCE%';
    l_result.message := replace(l_result.message, '%REFERENCE%', l_reference);

    -- Optional: Log the CGI values to see the browser and other info

    logger.log_apex_items(
      p_text => 'APEX unhandled exception (see logger_logs_apex_items)',
      p_scope => l_reference,
      p_level => logger.g_error);
    logger.log_error(p_error.message, l_reference, p_error.additional_info, l_params);

  end p_unhandled_exception;

begin
  -- Note: in future versions of Logger logging these values will be done in a single call
  logger.append_param(l_params, 'message', p_error.message);
  logger.append_param(l_params, 'additional_info', p_error.additional_info);
  logger.append_param(l_params, 'display_location', p_error.display_location);
  logger.append_param(l_params, 'association_type', p_error.association_type);
  logger.append_param(l_params, 'page_item_name', p_error.page_item_name);
  logger.append_param(l_params, 'region_id', p_error.region_id);
  logger.append_param(l_params, 'column_alias', p_error.column_alias);
  logger.append_param(l_params, 'row_num', p_error.row_num);
  logger.append_param(l_params, 'is_internal_error', p_error.is_internal_error);
  logger.append_param(l_params, 'apex_error_code', p_error.apex_error_code);
  logger.append_param(l_params, 'ora_sqlcode', p_error.ora_sqlcode);
  logger.append_param(l_params, 'ora_sqlerrm', p_error.ora_sqlerrm);
  logger.append_param(l_params, 'error_backtrace', p_error.error_backtrace);
  logger.append_param(l_params, 'component.type', p_error.component.type);
  logger.append_param(l_params, 'component.id', p_error.component.id);
  logger.append_param(l_params, 'component.name', p_error.component.name);
  logger.log('START', l_scope, null, l_params);

  l_result := apex_error.init_error_result (
                  p_error => p_error );

  -- If it's an internal error raised by APEX, like an invalid statement or
  -- code which cannot be executed, the error text might contain security sensitive
  -- information. To avoid this security problem rewrite the error to
  -- a generic error message and log the original error message for further
  -- investigation by the help desk.

 if p_error.is_internal_error then
    -- mask all errors that are not common runtime errors (Access Denied
    -- errors raised by application / page authorization and all errors
    -- regarding session and session state)
    if not p_error.is_common_runtime_error then

      p_unhandled_exception;

      l_result.additional_info := null;
    end if;

  else
    -- Always show the error as inline error
    -- Note: If you have created manual tabular forms (using the package
    --       apex_item/htmldb_item in the SQL statement) you should still
    --       use "On error page" on that pages to avoid loosing entered data
    l_result.display_location := case
                                   when l_result.display_location = apex_error.c_on_error_page then apex_error.c_inline_in_notification
                                   else l_result.display_location
                                 end;

    -- If an ORA error has been raised, for example a raise_application_error(-20xxx, '...')
    -- in a table trigger or in a PL/SQL package called by a process and the
    -- error has not been found in the lookup table, then display
    -- the actual error text and not the full error stack with all the ORA error numbers.
    if p_error.ora_sqlcode is not null and l_result.message = p_error.message then
      l_result.message := apex_error.get_first_ora_error_text (
                              p_error => p_error );
      p_unhandled_exception;
    end if;

    -- If no associated page item/tabular form column has been set, use
    -- apex_error.auto_set_associated_item to automatically guess the affected
    -- error field by examine the ORA error for constraint names or column names.
    if l_result.page_item_name is null and l_result.column_alias is null then
        apex_error.auto_set_associated_item (
            p_error        => p_error,
            p_error_result => l_result );
    end if;
  end if;

  logger.log('END', l_scope);
  return l_result;
end apex_error_handling_example;
/
```

A few things to note about this function:

- It will generate a generic error code based on the application ID and the date. You can use a sequence or any other technique to produce a unique code.
- You can modify as necessary to add any additional information. An example is to send an email notice when an error occurs.

In the APEX application register the error handling function:

- Shared Components
- Application Definition Attributes
- Error Handling Function: `apex_error_handling_example`
  - Apply Changes

Trigger error:

- Edit an employee
- Click on the `Trigger Error` button

Using the error code displayed on the screen, search the logs to try and find where the error occurred in the APEX application.

**Followup**

This exercise was a simple situation however it simulates what can happen in production. You may receive a screen shot with an error code and need to trace backwards from there.
