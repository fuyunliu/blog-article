# Sqlalchemy 基本用法

## 通用导入

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import scoped_session, sessionmaker
from sqlalchemy import Column, Integer, String, ForeignKey, Boolean
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base

engine = create_engine('sqlite:///test.db', echo=True)
Base = declarative_base()
db_session = scoped_session(sessionmaker(bind=engine))
Base.query = db_session.query_property()
```

## 一对一

```python
class Parent(Base):
    __tablename__ = 'parent'

    id = Column(Integer, primary_key=True)
    name = Column(String)
    child_id = Column(Integer, ForeignKey('child.id'))


class Child(Base):
    __tablename__ = 'child'

    id = Column(Integer, primary_key=True)
    name = Column(String)
    parent = relationship('Parent', backref='child', uselist=False)
```

## 一对多

```python
# the one side
class Parent(Base):
    __tablename__ = 'parent'

    id = Column(Integer, primary_key=True)
    name = Column(String)
    # children = relationship("Child", back_populates="parent")
    children = relationship("Child", backref="parent", lazy="dynamic")


# the many side
class Child(Base):
    __tablename__ = 'child'

    id = Column(Integer, primary_key=True)
    name = Column(String)
    parent_id = Column(Integer, ForeignKey('parent.id'))
    # parent = relationship("Parent", back_populates="children")
    # parent = relationship("Parent", backref="children")
```

## 多对一

```python
# the many side
class Parent(Base):
    __tablename__ = 'parent'

    id = Column(Integer, primary_key=True)
    name = Column(String)
    child_id = Column(Integer, ForeignKey('child.id'))


# the one side
class Child(Base):
    __tablename__ = 'child'

    id = Column(Integer, primary_key=True)
    name = Column(String)
    parents = relationship('Parent', backref='child', lazy='dynamic')
```

## 多对多

```python
class Department(Base):
    __tablename__ = 'department'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    employees = relationship(
        'Employee',
        secondary='department_employee_link'
    )


class Employee(Base):
    __tablename__ = 'employee'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    hired_on = Column(
        DateTime,
        default=func.now())
    departments = relationship(
        'Department',
        secondary='department_employee_link'
    )


class DepartmentEmployeeLink(Base):
    __tablename__ = 'department_employee_link'
    department_id = Column(Integer, ForeignKey('department.id'),
                           primary_key=True)
    department = relationship('Department')
    employee_id = Column(Integer, ForeignKey('employee.id'), primary_key=True)
    employee = relationship('Employee')
```

## 自身多对多

```python
class Follow(Base):
    __tablename__ = 'me_follow_you'

    me_id = Column(Integer, ForeignKey('users.id'), primary_key=True)
    me = relationship('User', foreign_keys=[me_id])
    you_id = Column(Integer, ForeignKey('users.id'), primary_key=True)
    you = relationship('User', foreign_keys=[you_id])
    created = Column(DateTime(timezone=True), default=datetime.utcnow)


class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String(64))

    # stars=我关注的人 fans=我的粉丝
    stars = relationship('User',
                         secondary='me_follow_you',
                         primaryjoin='User.id==Follow.me_id',
                         secondaryjoin='User.id==Follow.you_id',
                         backref=backref('fans', lazy='dynamic'),
                         lazy='dynamic')
```

## 自身一对一

```python
class Node(Base):
    __tablename__ = 'node'
    id = Column(Integer, primary_key=True)
    parent_id = Column(Integer, ForeignKey('node.id'))
    data = Column(String(50))
    parent = relationship("Node", remote_side=[id])
```

## 创建表

```python
def init_db():
    from sqlalchemy import create_engine
    engine = create_engine('sqlite:///test.db', echo=True)
    Base.metadata.create_all(engine)
```

## backref 和 back_populates

1. Parent 下添加 `children = relationship("Child", back_populates="parent")`

创建p1 = Parent()和c1 = Child()失败，原因是One or more mappers failed to initialize，即back_populates必须在关系两端同时指定

2. Parent下添加 `children = relationship("Child", back_populates="parent")`
   Child下添加 `parent = relationship("Parent", back_populates="children")`

Parent Attribute:
Parent.children Parent.id Parent.metadata Parent.name Parent.query

Child Attribute:
Child.id Child.metadata Child.name Child.parent Child.parent_id Child.query

p1 = Parent()
c1 = Child()
c1.parent = p1 or p1.children.append(c1)

3. Parent下添加 `children = relationship("Child", backref="parent")`

Parent Attribute:
Parent.children Parent.id Parent.metadata Parent.name Parent.query

Child Attribute:
Child.id Child.metadata Child.name Child.parent_id Child.query

p1 = Parent()
c1 = Child()
c1.parent = p1 or p1.children.append(c1)

可以看出使用backref时，实例化c1时会自动在c1对象上添加parent属性

此后再检查:
hasattr(Child, 'parent') // True
hasattr(c1, 'parent') // True
hasattr(Parent, 'children') // True
hasattr(p1, 'children') // True

4. Child 下添加 `parent = relationship("Parent", backref="children")` 情况和 3 相同

5. Parent下添加 `children = relationship("Child", backref="parent")`
   Child下添加 `parent = relationship("Parent", backref="children")`

创建p1 = Parent()和c1 = Child()失败，原因是One or more mappers failed to initialize
因此两者只能使用其中之一


lazy 指定如何加载相关记录，默认值是"select"
    select 首次访问时按需加载
    immediate 源对象加载后就加载
    joined 加载记录,但使用联结
    subquery 立即加载,但使用子查询
    noload 永不加载
    dynamic 不加载记录,但提供加载记录的查询

lazy = "dynamic"只能用于collections，不立即查询出结果集，而是提供一系列结果集的方法，可以基于结果集再次进行更精确的查找

## default 和 server_default

1. default 是在 ORM 层设置默认值，server_default 是在表结构上设置默认值
2. onupdate 在 ORM 层生效，server_onupdate 在数据库生效，在 MySQL 上 ON UPDATE 是MySQL在背后创建了 trigger，而在 PostgreSQL 上你必须手动创建 trigger

```python
from datetime import datetime
from sqlalchemy import func, sql, text

class Record(Base):
    __tablename__ = 'records
    
    id = Column(Integer, primary_key=True)
    name = Column(String(64), server_default=text('name'))
    created = Column(DateTime(timezone=True), default=datetime.utcnow)
    # created = Column(DateTime(timezone=True), server_default=func.now())
    # created = Column(DateTime(timezone=True), server_default=func.current_timestamp())
    updated = Column(DateTime(timezone=True), server_default=func.current_timestamp(), onupdate=func.current_timestamp())
    deleted = Column(Boolean, default=False)
    # deleted = Column(Boolean, server_default=sql.expression.false())
```

## 为flask_sqlalchemy扩展BaseQuery方法

```python
from flask_sqlalchemy import SQLAlchemy, BaseQuery
from sqlalchemy import func


class CustomQuery(BaseQuery):
    def count_all(self):
        return self.with_entities(func.count()).scalar()


db = SQLAlchemy(query_class=CustomQuery)
```



















