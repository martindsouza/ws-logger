# Logger Workshop: Procedures and Packages

As part of the [Best Practices](https://github.com/OraOpenSource/Logger/blob/master/docs/Best%20Practices.md) there's a code snippet that is recommended to use for packages and procedures. This section will cover how to structure a package to use Logger as well as some of the standard functions an tools that will be useful.

## Package Template

The following template is from the [Best Practices](https://github.com/OraOpenSource/Logger/blob/master/docs/Best%20Practices.md) and highlights how a package body should be structured. Note, comments have been removed for space purposes.

```sql
create or replace package body pkg_example
as

  gc_scope_prefix constant varchar2(31) := lower($$plsql_unit) || '.';

  procedure todo_proc_name(
    p_param1_todo in varchar2)
  as
    l_scope logger_logs.scope%type := gc_scope_prefix || 'todo_proc_name';
    l_params logger.tab_param;

  begin
    logger.append_param(l_params, 'p_param1_todo', p_param1_todo);
    logger.log('START', l_scope, null, l_params);

    ...
    -- All calls to logger should pass in the scope
    ...

    logger.log('END', l_scope);
  exception
    when others then
      logger.log_error('Unhandled Exception', l_scope, null, l_params);
      raise;
  end todo_proc_name;


end pkg_example;
/
```

A few things to highlight:

 Name | Description
--- | ---
`gc_scope_prefix` | This will be set the the `package_name.` and used for the scope (read below)
`l_scope` | Scoping is used to help filter and group logs. If no scope is provided it can be very difficult find what you're looking for. The recommended convention to use is `package_name.procedure_name`. Most Logger methods take in `p_scope` as a second parameter.
`l_params` | This is an array of parameters and is very useful when an error occurs. Most logging platforms just stores where the error occurs. By using the parameter array we also know what parameters cause the error.
`logger.append_param` | You need to explicitly append each parameter to the array. The good thing is that the procedure is overloaded so variables such as dates and booleans will automatically be converted to strings.


## Exercise

Create a package called `pkg_emp` with the following procedure:

`update_emp`

Parameter | Description
--- | ---
`p_empno` | Employee number
`p_sal` | Salary
`p_hiredate` | Hiredate

This procedure should update the appropriate `emp` record based on `p_empno`.

- Be sure to apply the best practices template listed above.
- Put a check in to verify that the `emp` exists. If it does not then an error should be raised.

Once the package is completed run the following code:

```sql
declare
begin
  pkg_emp.update_emp(p_empno => 7369, p_sal => 123, p_hiredate => trunc(sysdate));
end;
/
```

Query Logger to see all the information that is stored (especially the last, `extra`, column). Hint remember `logger_logs_5_min`.


Next, run the following code. Note that `empno` `9999` does not exist and an error should be raised.

```sql
declare
begin
  pkg_emp.update_emp(p_empno => 9999, p_sal => 123, p_hiredate => trunc(sysdate));
end;
/
```

Query Logger again and see the results. This time, filter on the `scope` to only see content for the package.

Suppose that you didn't write the package. Given the log information can you find out what caused the issue and where the error occurred?


## Summary

Following this section you should be familiar with the following:

- Using the template to write procedures and functions
- Understand parameters, scopes, and where they are stored in `logger_logs`
- Given an error, debug where the error occurred and what parameters caused it.
