---
title: 牛客网sql练习
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["sql"]
tags: ["sql"]
---

# 牛客网sql练习
#sql
1. 查找employees里最晚入职员工的所有信息
```sql
select * from employees order by hire_date desc;
```

2. 查找employees里入职员工时间排名倒数第三的员工所有信息
```sql
select * from employees order by hire_date desc limit 1 offset 2;
```
> limit 表示最终结果需要多少个，offset表示跳过多少个结果。本题倒序排序后，offset 2表示跳过前两个，因此下一个就是倒数第三个，limit 1就取了这个

```sql
select * from employees order by hire_date desc limit 2,1;
```
> limit 第一个参数表示跳过多少个，第二个参数表示选取多少个，因此跳过前两个，选取下一个，即为答案

3. 查找各个部门领导薪水详情以及其对应部门编号dept_no，输出结果以salaries.emp_no升序排序，并且请注意输出结果里面dept_no列是最后一列
```sql
select d.emp_no,s.salary,d.dept_no from dept_manager as d left join salaries as s on d.emp_no = s.emp_no order by emp_no asc;
```
> 需要查找领导的薪水详情，所以必须是领导表里有的，因此对领导表做左连接查询，左连接的条件是领导表与薪水表的编号相同

4. 查找所有已经分配部门的员工的last_name和first_name以及dept_no，未分配的部门的员工不显示
```sql
select e.last_name, e.first_name, d.dept_no from dept_emp as d left join employees as e on d.emp_no = e.emp_no;
```
> 由于要查找已分配部门的员工，部门表的所有记录都是分配了部门的员工，因此针对部门表做左连接即可，同时连接员工表，取出名字信息，以员工号作为连接条件即可

5. 查找所有已经分配部门的员工的last_name和first_name以及dept_no，也包括暂时没有分配具体部门的员工
```sql
select e.last_name, e.first_name, d.dept_no from employees as e left join dept_emp as d on e.emp_no = d.emp_no;
```
> 本题跟上题的区别是也要查出为分配部门的员工，因此员工表是基准，对员工表做左连接，部门表中不存在的记录为NULL

6. 查找薪水记录超过2次的员工号emp_no以及其对应的记录次数t
```sql
select emp_no, count(*) as t from salaries group by emp_no having t > 2;
```
> 薪水记录超过15次，因此需要count，通过emp_no分组，每一组的结果即为该员工的薪水记录数。having关键字是筛选成组后的数据，where关键字是在聚合前筛选记录。

7. 找出所有非部门领导的员工emp_no
```sql
select e.emp_no from employees as e left join dept_manager as d on e.emp_no = d.emp_no where d.emp_no is NULL;
```
> 通过员工表与领导表左连接，员工表在领导表中不存在的记录会是NULL，因此排除掉NULL的字段值即可

8. 获取所有的员工和员工对应的经理，如果员工本身是经理的话则不显示
```sql
select de.emp_no, dm.emp_no as manager from dept_emp as de inner join dept_manager as dm on de.dept_no = dm.dept_no where de.emp_no != dm.emp_no;
```
> 员工和员工对应的领导是在一个部门的，因此对部门员工表和领导表进行关联。部门员工与领导有相同的部门，因此做内连接。因为员工本身是领导不展示，因此排除掉部门员工与领导表工号相同的记录。
