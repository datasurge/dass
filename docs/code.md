#### Model Feature code

以下实现模型功能的增删改查：

```python
import pymysql
pymysql.install_as_MySQLdb()
from .handlers import setup_handler
def _jupyter_server_extension_paths():
    '''
     Declare the Jupyter server extension paths.
    '''
    return [{"module": "jupyterlab_model"}]

def _jupyter_nbextension_paths():
    """
    Declare the Jupyter notebook extension paths.
    :return:
    """
    return [{'section': "notebook", "dest": "jupyterlab_model"}]

def load_jupyter_server_extension(nbapp):
    """
    Load the Jupyter server extension.
    """
    setup_handler(nbapp.web_app)
```

```python
# 配置数据库
import os
DB_HOST =  os.getenv('DB_HOST') or '192.168.3.51'
DB_PORT = os.getenv('DB_PORT') or 3306
DB_USER = os.getenv("DB_USER") or 'root'
DB_PASSWD = os.getenv("DB_PASSWD") or 'yhds2'
DB_NAME = os.getenv("DB_NAME") or 'dass'
DB_CHARTSET = os.getenv("DB_CHARTSET") or 'utf8'
```

```python
# 连接数据库，并创建表

import sqlalchemy
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String, ForeignKey

from .conf import DB_HOST, DB_NAME, DB_PASSWD, DB_USER, DB_CHARTSET

database = 'mysql://%s:%s@%s/%s?%s' % (DB_USER, DB_PASSWD, DB_HOST, DB_NAME, DB_CHARTSET)

engine = sqlalchemy.create_engine(database, encoding='utf-8', echo=True)
Session = sessionmaker(bind=engine)
session = Session()

base = declarative_base()

class User(base):

    __tablename__ = 'users'

    UID = Column(Integer, primary_key=True, autoincrement=True, comment=u'用户ID')
    Foreign_ID = Column(Integer, comment=u'映射ID')

    # Users = relationship('UserModelType')

    def __str__(self):
        return '%s' % self.UID

class Model(base):

    __tablename__ = 'models'

    MID = Column(Integer, primary_key=True, comment=u'模型ID', autoincrement=True)
    Model_Name = Column(String(128), comment=u'模型类名')
    Code = Column(String(2048), comment=u'代码存储')
    Permission = Column(Integer, default=1, comment=u'权限认证')
    Desc = Column(String(128), comment=u'详细信息')

    def __str__(self):
        return self.Model_Name

class Type(base):

    __tablename__ = 'types_model'

    TID = Column(Integer, primary_key=True, autoincrement=True, comment=u'类型ID')
    Type_name = Column(String(32), nullable=False, comment=u'类型名')
    Father_ID = Column(Integer, comment=u'父类ID')

    def __str__(self):
        return self.Type_name

class User_Model_Type(base):

    __tablename__ = 'user_model_type'

    ID = Column(Integer, primary_key=True, autoincrement=True)
    User_ID = Column(Integer, ForeignKey('users.UID'), comment=u'用户ID', nullable=False, primary_key=True)
    Model_ID = Column(Integer, ForeignKey("models.MID"), comment=u'模型ID', nullable=False, primary_key=True)
    Type_ID = Column(Integer, ForeignKey('types_model.TID'), comment=u'类型ID', nullable=False, primary_key=True)


    def __str__(self):
        return "User_Model_Type(user=%s,model=%s,type=%s)" % (self.User_ID, self.Model_ID, self.Type_ID)

def create_DB():
    print(engine)
    base.metadata.create_all(engine)
```

```python
# 增加模型接口
from sqlalchemy.orm import sessionmaker
from notebook.base.handlers import APIHandler

from wtforms.fields import IntegerField, StringField
from wtforms.validators import Required
from wtforms_tornado import Form

from .db_link import Model, engine

class TypesForm(Form):
    model_name = StringField(validators=[Required])
    code = StringField(validators=[Required])
    permission = IntegerField(validators=[Required])
    desc = StringField(validators=[Required])

class Add_models(APIHandler):
    """
    定义添加models类
    """

    def initialize(self):
        session = sessionmaker(bind=engine)
        self.session = session()

    def get(self, *args, **kwargs):
        pass

    def post(self, *args, **kwargs):
        """
        前端获取数据,执行sql语句保存数据库
        """
        form = TypesForm(self.request.arguments)
        if form.validate():
            models = Model()
            models.Model_Name = form.data['model_name']
            models.Code = form.data['code']
            models.Permission = form.data['permission']
            models.Desc = form.data['desc']
            self.session.add(models)
            self.session.commit()
            return self.write('Save OK')
        else:
            return self.write(form.errors)

add_handler = r'/model/add'
```

```python
# 获取初始类型信息
from .db_link import User_Model_Type, Model, Type, engine
from sqlalchemy.orm import sessionmaker
# from tornado.web import RequestHandler
from notebook.base.handlers import APIHandler
from tornado import web

class Type_Handler(APIHandler):

    def initialize(self):
        sess = sessionmaker(bind=engine)
        self.db = sess()

    @web.authenticated
    def get(self, *args, **kwargs):
        t_dict = {}
        # Get information about the type table
        types = self.db.query(Type)
        f_list = types.filter(Type.Father_ID == None)

        t_list = []
        for f_type in f_list:
            # Place the subset under the corresponding parent
            # Get the subset information corresponding to each parent ID
            child_list = types.filter(Type.Father_ID == f_type.TID)
            # Subset information list
            child_ = []
            for child_type in child_list:

                # Mode description list
                model_list = []
                # Get the current type of model information, ID, limited to ten
                models = self.db.query(User_Model_Type).filter(User_Model_Type.Type_ID == child_type.TID).limit(10)
                for model in models:
                    m_id = model.Model_ID
                    # Get the corresponding model information
                    m_desc = self.db.query(Model).filter(Model.MID == m_id).first()
                    model_list.append({'mid': m_desc.MID, 'mname': m_desc.Model_Name})
                # Add a model list to the subtype list
                child_.append({'cid': child_type.TID, 'cname': child_type.Type_name,'mdesc':model_list})
            # Add a speaker type to the parent class list
            t_list.append({'father_id': f_type.TID, 'type_name': f_type.Type_name, 'child': child_})
        t_dict['type'] = t_list
        # result data
        return self.write(t_dict)

type_handler = r'/model/type'

```

```python
#! -*- encoding: utf-8 -*-
# 注册增加的模型处理路由
from notebook.utils import url_path_join as ujoin

from .add_handler import Add_models, add_handler
from .type_handler import Type_Handler, type_handler

def setup_handler(web_app):
    pass
    '''
    Setup all of the model command handlers.
    '''
    model_hamdler = [
            (add_handler, Add_models),
            (type_handler, Type_Handler)
        ]

    base_url = web_app.settings['base_url']
    model_hamdlers = [(ujoin(base_url, x[0]), x[1]) for x in model_hamdler]

    web_app.add_handlers(".*", model_hamdlers)
```

#### Git Feature code

```python
"""
Initialize the backend server extension
"""
from jupyterlab_git.handlers import setup_handlers
from jupyterlab_git.git import Git

def _jupyter_server_extension_paths():
    """
    Declare the Jupyter server extension paths.
    """
    return [{"module": "jupyterlab_git"}]

def _jupyter_nbextension_paths():
    """
    Declare the Jupyter notebook extension paths.
    """
    return [{"section": "notebook", "dest": "jupyterlab_git"}]

def load_jupyter_server_extension(nbapp):
    """
    Load the Jupyter server extension.
    """
    git = Git()
    nbapp.web_app.settings["git"] = git
    setup_handlers(nbapp.web_app)
```

```python
"""
Module for executing git commands, sending results back to the handlers
"""
import os
import subprocess
from subprocess import Popen, PIPE


class Git:
    """
    A single parent class containing all of the individual git methods in it.
    """

    def status(self, current_path):
        """
        Execute git status command & return the result.
        """
        p = Popen(
            ["git", "status", "--porcelain"],
            stdout=PIPE,
            stderr=PIPE,
            cwd=os.getcwd() + "/" + current_path,
        )
        my_output, my_error = p.communicate()
        if p.returncode == 0:
            result = []
            line_array = my_output.decode("utf-8").splitlines()
            for line in line_array:
                to1 = None
                from_path = line[3:]
                if line[0] == "R":
                    to0 = line[3:].split(" -> ")
                    to1 = to0[len(to0) - 1]
                else:
                    to1 = line[3:]
                if to1.startswith('"'):
                    to1 = to1[1:]
                if to1.endswith('"'):
                    to1 = to1[:-1]
                result.append(
                    {"x": line[0], "y": line[1], "to": to1, "from": from_path}
                )
            return {"code": p.returncode, "files": result}
        else:
            return {
                "code": p.returncode,
                "command": "git status --porcelain",
                "message": my_error.decode("utf-8"),
            }

    def log(self, current_path):
        """
        Execute git log command & return the result.
        """
        p = Popen(
            ["git", "log", "--pretty=format:%H%n%an%n%ar%n%s", "-10"],
            stdout=PIPE,
            stderr=PIPE,
            cwd=os.getcwd() + "/" + current_path,
        )
        my_output, my_error = p.communicate()
        if p.returncode == 0:
            result = []
            line_array = my_output.decode("utf-8").splitlines()
            i = 0
            PREVIOUS_COMMIT_OFFSET = 4
            while i < len(line_array):
                if i + PREVIOUS_COMMIT_OFFSET < len(line_array):
                    result.append(
                        {
                            "commit": line_array[i],
                            "author": line_array[i + 1],
                            "date": line_array[i + 2],
                            "commit_msg": line_array[i + 3],
                            "pre_commit": line_array[i + PREVIOUS_COMMIT_OFFSET],
                        }
                    )
                else:
                    result.append(
                        {
                            "commit": line_array[i],
                            "author": line_array[i + 1],
                            "date": line_array[i + 2],
                            "commit_msg": line_array[i + 3],
                            "pre_commit": "",
                        }
                    )
                i += PREVIOUS_COMMIT_OFFSET
            return {"code": p.returncode, "commits": result}
        else:
            return {"code": p.returncode, "message": my_error.decode("utf-8")}

    def detailed_log(self, selected_hash, current_path):
        """
        Execute git log -1 --stat --numstat --oneline command (used to get
        insertions & deletions per file) & return the result.
        """
        p = Popen(
            ["git", "log", "-1", "--stat", "--numstat", "--oneline", selected_hash],
            stdout=PIPE,
            stderr=PIPE,
            cwd=os.getcwd() + "/" + current_path,
        )
        my_output, my_error = p.communicate()
        if p.returncode == 0:
            result = []
            note = [0] * 3
            count = 0
            temp = ""
            line_array = my_output.decode("utf-8").splitlines()
            length = len(line_array)
            INSERTION_INDEX = 0
            DELETION_INDEX = 1
            MODIFIED_FILE_PATH_INDEX = 2
            if length > 1:
                temp = line_array[length - 1]
                words = temp.split()
                for i in range(0, len(words)):
                    if words[i].isdigit():
                        note[count] = words[i]
                        count += 1
                for num in range(1, int(length / 2)):
                    line_info = line_array[num].split()
                    words = line_info[2].split("/")
                    length = len(words)
                    result.append(
                        {
                            "modified_file_path": line_info[MODIFIED_FILE_PATH_INDEX],
                            "modified_file_name": words[length - 1],
                            "insertion": line_info[INSERTION_INDEX],
                            "deletion": line_info[DELETION_INDEX],
                        }
                    )

            if note[2] == 0 and length > 1:
                if "-" in temp:
                    exchange = note[1]
                    note[1] = note[2]
                    note[2] = exchange

            return {
                "code": p.returncode,
                "modified_file_note": temp,
                "modified_files_count": note[0],
                "number_of_insertions": note[1],
                "number_of_deletions": note[2],
                "modified_files": result,
            }
        else:
            return {
                "code": p.returncode,
                "command": "git log_1",
                "message": my_error.decode("utf-8"),
            }

    def diff(self, top_repo_path):
        """
        Execute git diff command & return the result.
        """
        p = Popen(
            ["git", "diff", "--numstat"], stdout=PIPE, stderr=PIPE, cwd=top_repo_path
        )
        my_output, my_error = p.communicate()
        if p.returncode == 0:
            result = []
            line_array = my_output.decode("utf-8").splitlines()
            for line in line_array:
                linesplit = line.split()
                result.append(
                    {
                        "insertions": linesplit[0],
                        "deletions": linesplit[1],
                        "filename": linesplit[2],
                    }
                )
            return {"code": p.returncode, "result": result}
        else:
            return {"code": p.returncode, "message": my_error.decode("utf-8")}

    def branch(self, current_path):
        """
        Execute git branch -a command & return the result.
        """
        p = Popen(
            ["git", "branch", "-a"],
            stdout=PIPE,
            stderr=PIPE,
            cwd=os.getcwd() + "/" + current_path,
        )
        my_output, my_error = p.communicate()
        if p.returncode == 0:
            result = []
            line_array = my_output.decode("utf-8").splitlines()
            """By comparing strings 'remotes/' to determine if a branch is
            local or remote, should have better ways
            """
            for line_full in line_array:
                line_cut = (line_full.split(" -> "),)
                tag = None
                current = False
                remote = False
                if len(line_cut[0]) > 1:
                    tag = line_cut[0][1]
                line = (line_cut[0][0],)
                if line_full[0] == "*":
                    current = True
                if (len(line_full) >= 10) and (line_full[2:10] == "remotes/"):
                    remote = True
                    result.append(
                        {
                            "current": current,
                            "remote": remote,
                            "name": line[0][10:],
                            "tag": tag,
                        }
                    )
                else:
                    result.append(
                        {
                            "current": current,
                            "remote": remote,
                            "name": line_full[2:],
                            "tag": tag,
                        }
                    )
            return {"code": p.returncode, "branches": result}
        else:
            return {
                "code": p.returncode,
                "command": "git branch -a",
                "message": my_error.decode("utf-8"),
            }

    def show_top_level(self, current_path):
        """
        Execute git --show-toplevel command & return the result.
        """
        p = Popen(
            ["git", "rev-parse", "--show-toplevel"],
            stdout=PIPE,
            stderr=PIPE,
            cwd=os.getcwd() + "/" + current_path,
        )
        my_output, my_error = p.communicate()
        if p.returncode == 0:
            result = {
                "code": p.returncode,
                "top_repo_path": my_output.decode("utf-8").strip("\n"),
            }
            return result
        else:
            return {
                "code": p.returncode,
                "command": "git rev-parse --show-toplevel",
                "message": my_error.decode("utf-8"),
            }

    def show_prefix(self, current_path):
        """
        Execute git --show-prefix command & return the result.
        """
        p = Popen(
            ["git", "rev-parse", "--show-prefix"],
            stdout=PIPE,
            stderr=PIPE,
            cwd=os.getcwd() + "/" + current_path,
        )
        my_output, my_error = p.communicate()
        if p.returncode == 0:
            result = {
                "code": p.returncode,
                "under_repo_path": my_output.decode("utf-8").strip("\n"),
            }
            return result
        else:
            return {
                "code": p.returncode,
                "command": "git rev-parse --show-prefix",
                "message": my_error.decode("utf-8"),
            }

    def add(self, filename, top_repo_path):
        """
        Execute git add<filename> command & return the result.
        """
        my_output = subprocess.check_output(["git", "add", filename], cwd=top_repo_path)
        return my_output

    def add_all(self, top_repo_path):
        """
        Execute git add all command & return the result.
        """
        my_output = subprocess.check_output(["git", "add", "-A"], cwd=top_repo_path)
        return my_output

    def add_all_untracked(self, top_repo_path):
        """
        Execute git add_all_untracked command & return the result.
        """
        e = 'echo "a\n*\nq\n" | git add -i'
        my_output = subprocess.call(e, shell=True, cwd=top_repo_path)
        return {"result": my_output}

    def reset(self, filename, top_repo_path):
        """
        Execute git reset <filename> command & return the result.
        """
        my_output = subprocess.check_output(
            ["git", "reset", filename], cwd=top_repo_path
        )
        return my_output

    def reset_all(self, top_repo_path):
        """
        Execute git reset command & return the result.
        """
        my_output = subprocess.check_output(["git", "reset"], cwd=top_repo_path)
        return my_output

    def delete_commit(self, commit_id, top_repo_path):
        """
        Delete a specified commit from the repository.
        """
        my_output = subprocess.check_output(["git", "revert", "--no-commit", commit_id], cwd=top_repo_path)
        return my_output

    def reset_to_commit(self, commit_id, top_repo_path):
        """
        Reset the current branch to a specific past commit.
        """
        my_output = subprocess.check_output(["git", "reset", "--hard", commit_id], cwd=top_repo_path)
        return my_output

    def checkout_new_branch(self, branchname, current_path):
        """
        Execute git checkout <make-branch> command & return the result.
        """
        p = Popen(
            ["git", "checkout", "-b", branchname],
            stdout=PIPE,
            stderr=PIPE,
            cwd=os.getcwd() + "/" + current_path,
        )
        my_output, my_error = p.communicate()
        if p.returncode == 0:
            return {"code": p.returncode, "message": my_output.decode("utf-8")}
        else:
            return {
                "code": p.returncode,
                "command": "git checkout " + "-b" + branchname,
                "message": my_error.decode("utf-8"),
            }

    def checkout_branch(self, branchname, current_path):
        """
        Execute git checkout <branch-name> command & return the result.
        """
        p = Popen(
            ["git", "checkout", branchname],
            stdout=PIPE,
            stderr=PIPE,
            cwd=os.getcwd() + "/" + current_path,
        )
        my_output, my_error = p.communicate()
        if p.returncode == 0:
            return {"code": p.returncode, "message": my_output.decode("utf-8")}
        else:
            return {
                "code": p.returncode,
                "command": "git checkout " + branchname,
                "message": my_error.decode("utf-8"),
            }

    def checkout(self, filename, top_repo_path):
        """
        Execute git checkout command for the filename & return the result.
        """
        my_output = subprocess.check_output(
            ["git", "checkout", "--", filename], cwd=top_repo_path
        )
        return my_output

    def checkout_all(self, top_repo_path):
        """
        Execute git checkout command & return the result.
        """
        my_output = subprocess.check_output(
            ["git", "checkout", "--", "."], cwd=top_repo_path
        )
        return my_output

    def commit(self, commit_msg, top_repo_path):
        """
        Execute git commit <filename> command & return the result.
        """
        my_output = subprocess.check_output(
            ["git", "commit", "-m", commit_msg], cwd=top_repo_path
        )
        return my_output

    def pull(self, origin, master, curr_fb_path):
        """
        Execute git pull <branch1> <branch2> command & return the result.
        """
        p = Popen(
            ["git", "pull", origin, master, "--no-commit"],
            stdout=PIPE,
            stderr=PIPE,
            cwd=os.getcwd() + "/" + curr_fb_path,
        )
        my_output, my_error = p.communicate()
        if p.returncode == 0:
            return {"code": p.returncode}
        else:
            return {
                "code": p.returncode,
                "command": "git pull " + origin + " " + master + " --no-commit",
                "message": my_error.decode("utf-8"),
            }

    def push(self, origin, master, curr_fb_path):
        """
        Execute git push <branch1> <branch2> command & return the result.
        """
        p = Popen(
            ["git", "push", origin, master],
            stdout=PIPE,
            stderr=PIPE,
            cwd=os.getcwd() + "/" + curr_fb_path,
        )
        my_output, my_error = p.communicate()
        if p.returncode == 0:
            return {"code": p.returncode}
        else:
            return {
                "code": p.returncode,
                "command": "git push " + origin + " " + master,
                "message": my_error.decode("utf-8"),
            }

    def init(self, current_path):
        """
        Execute git init command & return the result.
        """
        my_output = subprocess.check_output(
            ["git", "init"], cwd=os.getcwd() + "/" + current_path
        )
        return my_output
```

```python
import json


from notebook.utils import url_path_join as ujoin
from notebook.base.handlers import APIHandler


class GitHandler(APIHandler):
    """
    Top-level parent class.
    """

    @property
    def git(self):
        return self.settings["git"]


class GitAllHistoryHandler(GitHandler):
    """
    Parent handler for all four history/status git commands:
    1. git show_top_level
    2. git branch
    3. git log
    4. git status
    Called on refresh of extension's widget
    """

    def post(self):
        """
        POST request handler, calls individual handlers for 
        'git show_top_level', 'git branch', 'git log', and 'git status'
        """
        data = json.loads(self.request.body.decode("utf-8"))
        current_path = data["current_path"]
        show_top_level = self.git.show_top_level(current_path)
        if show_top_level["code"] != 0:
            self.finish(json.dumps(show_top_level))
        else:
            branch = self.git.branch(current_path)
            log = self.git.log(current_path)
            status = self.git.status(current_path)

            result = {
                "code": show_top_level["code"],
                "data": {
                    "show_top_level": show_top_level,
                    "branch": branch,
                    "log": log,
                    "status": status,
                },
            }
            self.finish(json.dumps(result))


class GitShowTopLevelHandler(GitHandler):
    """
    Handler for 'git rev-parse --show-toplevel'. 
    Displays the git root directory inside a repository.
    """

    def post(self):
        """
        POST request handler, displays the git root directory inside a repository.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        current_path = data["current_path"]
        result = self.git.show_top_level(current_path)
        self.finish(json.dumps(result))


class GitShowPrefixHandler(GitHandler):
    """
    Handler for 'git rev-parse --show-prefix'. 
    Displays the prefix path of a directory in a repository, 
    with respect to the root directory.
    """

    def post(self):
        """
        POST request handler, displays the prefix path of a directory in a repository, 
        with respect to the root directory.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        current_path = data["current_path"]
        result = self.git.show_prefix(current_path)
        self.finish(json.dumps(result))


class GitStatusHandler(GitHandler):
    """
    Handler for 'git status --porcelain', fetches the git status.
    """

    def get(self):
        """
        GET request handler, shows file status, used in refresh method.
        """
        self.finish(
            json.dumps(
                {"add_all": "check", "filename": "filename", "top_repo_path": "path"}
            )
        )

    def post(self):
        """
        POST request handler, fetches the git status.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        current_path = data["current_path"]
        result = self.git.status(current_path)
        self.finish(json.dumps(result))


class GitLogHandler(GitHandler):
    """
    Handler for 'git log --pretty=format:%H-%an-%ar-%s'. 
    Fetches Commit SHA, Author Name, Commit Date & Commit Message.
    """

    def post(self):
        """
        POST request handler, 
        fetches Commit SHA, Author Name, Commit Date & Commit Message.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        current_path = data["current_path"]
        result = self.git.log(current_path)
        self.finish(json.dumps(result))


class GitDetailedLogHandler(GitHandler):
    """
    Handler for 'git log -1 --stat --numstat --oneline' command. 
    Fetches file names of committed files, Number of insertions &
    deletions in that commit.
    """

    def post(self):
        """
        POST request handler, fetches file names of committed files, Number of
        insertions & deletions in that commit.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        selected_hash = data["selected_hash"]
        current_path = data["current_path"]
        result = self.git.detailed_log(selected_hash, current_path)
        self.finish(json.dumps(result))


class GitDiffHandler(GitHandler):
    """
    Handler for 'git diff --numstat'. Fetches changes between commits & working tree.
    """

    def post(self):
        """
        POST request handler, fetches differences between commits & current working
        tree.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        top_repo_path = data["top_repo_path"]
        my_output = self.git.diff(top_repo_path)
        self.finish(my_output)
        print("GIT DIFF")
        print(my_output)


class GitBranchHandler(GitHandler):
    """
    Handler for 'git branch -a'. Fetches list of all branches in current repository
    """

    def post(self):
        """
        POST request handler, fetches all branches in current repository.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        current_path = data["current_path"]
        result = self.git.branch(current_path)
        self.finish(json.dumps(result))


class GitAddHandler(GitHandler):
    """
    Handler for git add <filename>'. 
    Adds one or all files into to the staging area.
    """

    def get(self):
        """
        GET request handler, adds files in the staging area.
        """
        self.finish(
            json.dumps(
                {"add_all": "check", "filename": "filename", "top_repo_path": "path"}
            )
        )

    def post(self):
        """
        POST request handler, adds one or all files into the staging area.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        top_repo_path = data["top_repo_path"]
        if data["add_all"]:
            my_output = self.git.add_all(top_repo_path)
        else:
            filename = data["filename"]
            my_output = self.git.add(filename, top_repo_path)
        self.finish(my_output)


class GitResetHandler(GitHandler):
    """
    Handler for 'git reset <filename>'. 
    Moves one or all files from the staged to the unstaged area.
    """

    def post(self):
        """
        POST request handler, 
        moves one or all files from the staged to the unstaged area.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        top_repo_path = data["top_repo_path"]
        if data["reset_all"]:
            my_output = self.git.reset_all(top_repo_path)
        else:
            filename = data["filename"]
            my_output = self.git.reset(filename, top_repo_path)
        self.finish(my_output)

class GitDeleteCommitHandler(GitHandler):
    """
    Handler for 'git revert --no-commit <SHA>'.
    Deletes the specified commit from the repository, leaving history intact.
    """

    def post(self):
        data = json.loads(self.request.body.decode("utf-8"))
        top_repo_path = data["top_repo_path"]
        commit_id = data["commit_id"]
        output = self.git.delete_commit(commit_id, top_repo_path)
        self.finish(output)

class GitResetToCommitHandler(GitHandler):
    """
    Handler for 'git reset --hard <SHA>'.
    Deletes all commits from head to the specified commit, making the specified commit the new head.
    """

    def post(self):
        data = json.loads(self.request.body.decode("utf-8"))
        top_repo_path = data["top_repo_path"]
        commit_id = data["commit_id"]
        output = self.git.reset_to_commit(commit_id, top_repo_path)
        self.finish(output)

class GitCheckoutHandler(GitHandler):
    """
    Handler for 'git checkout <branchname>'. Changes the current working branch.
    """

    def post(self):
        """
        POST request handler, changes between branches.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        top_repo_path = data["top_repo_path"]
        if data["checkout_branch"]:
            if data["new_check"]:
                print("to create a new branch")
                my_output = self.git.checkout_new_branch(
                    data["branchname"], top_repo_path
                )
            else:
                print("switch to an old branch")
                my_output = self.git.checkout_branch(
                    data["branchname"], top_repo_path
                )
        elif data["checkout_all"]:
            my_output = self.git.checkout_all(top_repo_path)
        else:
            my_output = self.git.checkout(data["filename"], top_repo_path)
        self.finish(my_output)


class GitCommitHandler(GitHandler):
    """
    Handler for 'git commit -m <message>'. Commits files.
    """

    def post(self):
        """
        POST request handler, commits files.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        top_repo_path = data["top_repo_path"]
        commit_msg = data["commit_msg"]
        my_output = self.git.commit(commit_msg, top_repo_path)
        self.finish(my_output)


class GitPullHandler(GitHandler):
    """
    Handler for 'git pull <first-branch> <second-branch>'. Pulls files from a remote branch.
    """

    def post(self):
        """
        POST request handler, pulls files from a remote branch to your current branch.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        origin = data["origin"]
        master = data["master"]
        curr_fb_path = data["curr_fb_path"]
        my_output = self.git.pull(origin, master, curr_fb_path)
        self.finish(my_output)
        print("You Pulled")


class GitPushHandler(GitHandler):
    """
    Handler for 'git push <first-branch> <second-branch>. 
    Pushes committed files to a remote branch.
    """

    def post(self):
        """
        POST request handler, 
        pushes comitted files from your current branch to a remote branch
        """
        data = json.loads(self.request.body.decode("utf-8"))
        origin = data["origin"]
        master = data["master"]
        curr_fb_path = data["curr_fb_path"]
        my_output = self.git.push(origin, master, curr_fb_path)
        self.finish(my_output)
        print("You Pushed")


class GitInitHandler(GitHandler):
    """
    Handler for 'git init'. Initializes a repository.
    """

    def post(self):
        """
        POST request handler, initializes a repository.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        current_path = data["current_path"]
        my_output = self.git.init(current_path)
        self.finish(my_output)


class GitAddAllUntrackedHandler(GitHandler):
    """
    Handler for 'echo "a\n*\nq\n" | git add -i'. Adds ONLY all untracked files.
    """

    def post(self):
        """
        POST request handler, adds all the untracked files.
        """
        data = json.loads(self.request.body.decode("utf-8"))
        top_repo_path = data["top_repo_path"]
        my_output = self.git.add_all_untracked(top_repo_path)
        print(my_output)
        self.finish(my_output)


def setup_handlers(web_app):
    """
    Setups all of the git command handlers.
    Every handler is defined here, to be used in git.py file.
    """

    git_handlers = [
        ("/git/show_top_level", GitShowTopLevelHandler),
        ("/git/show_prefix", GitShowPrefixHandler),
        ("/git/add", GitAddHandler),
        ("/git/status", GitStatusHandler),
        ("/git/branch", GitBranchHandler),
        ("/git/reset", GitResetHandler),
        ("/git/delete_commit", GitDeleteCommitHandler),
        ("/git/reset_to_commit", GitResetToCommitHandler),
        ("/git/checkout", GitCheckoutHandler),
        ("/git/commit", GitCommitHandler),
        ("/git/pull", GitPullHandler),
        ("/git/push", GitPushHandler),
        ("/git/diff", GitDiffHandler),
        ("/git/log", GitLogHandler),
        ("/git/detailed_log", GitDetailedLogHandler),
        ("/git/init", GitInitHandler),
        ("/git/all_history", GitAllHistoryHandler),
        ("/git/add_all_untracked", GitAddAllUntrackedHandler),
    ]

    # add the baseurl to our paths
    base_url = web_app.settings["base_url"]
    git_handlers = [(ujoin(base_url, x[0]), x[1]) for x in git_handlers]
    print("base_url: {}".format(base_url))
    print(git_handlers)

    web_app.add_handlers(".*", git_handlers)
```

#### Lab Feature code

```python
"""Server extension for JupyterLab."""
# __init__.py
# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.

from ._version import __version__
from .extension import load_jupyter_server_extension

def _jupyter_server_extension_paths():
    return [{
        "module": "jupyterlab"
    }]
```

```python
# __main__.py
from jupyterlab.labapp import main
import sys

sys.exit(main())
```

```python
# _version.py
version_info = (0, 32, 0)
print("_version.py !!!!!!!!!!!!!!!!!!!")
__version__ = ".".join(map(str, version_info))
```

```python
# coding: utf-8
# labapp
"""A tornado based Jupyter lab server."""

# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.

from notebook.notebookapp import NotebookApp, aliases, flags
from jupyter_core.application import JupyterApp, base_aliases

from traitlets import Bool, Unicode

from ._version import __version__
from .extension import load_jupyter_server_extension
from .commands import (
    build, clean, get_app_dir, get_user_settings_dir, get_app_version,
    get_workspaces_dir
)

build_aliases = dict(base_aliases)
build_aliases['app-dir'] = 'LabBuildApp.app_dir'
build_aliases['name'] = 'LabBuildApp.name'
build_aliases['version'] = 'LabBuildApp.version'

build_flags = dict(flags)
build_flags['dev'] = (
    {'LabBuildApp': {'dev_build': True}},
    "Build in Development mode"
)

version = __version__
app_version = get_app_version()
if version != app_version:
    version = '%s (dev), %s (app)' % (__version__, app_version)


class LabBuildApp(JupyterApp):
    # def __init__(self):
    #　    print("#######################################Lab app.LabBuildApp")

    version = version
    description = """
    Build the JupyterLab application

    The application is built in the JupyterLab app directory in `/staging`.
    When the build is complete it is put in the JupyterLab app `/static`
    directory, where it is used to serve the application.
    """
    aliases = build_aliases
    flags = build_flags

    app_dir = Unicode('', config=True,
        help="The app directory to build in")

    name = Unicode('JupyterLab', config=True,
        help="The name of the built application")

    version = Unicode('', config=True,
        help="The version of the built application")

    dev_build = Bool(True, config=True,
        help="Whether to build in dev mode (defaults to production mode)")

    def start(self):
        command = 'build:prod' if not self.dev_build else 'build'
        build(app_dir=self.app_dir, name=self.name, version=self.version,
              command=command, logger=self.log)


clean_aliases = dict(base_aliases)
clean_aliases['app-dir'] = 'LabCleanApp.app_dir'


class LabCleanApp(JupyterApp):
    version = version
    description = """
    Clean the JupyterLab application

    This will clean the app directory by removing the `staging` and `static`
    directories.
    """
    aliases = clean_aliases

    app_dir = Unicode('', config=True,
        help="The app directory to clean")

    def start(self):
        clean(self.app_dir)


class LabPathApp(JupyterApp):

    # def __init__(self):
    #     print("#######################################Lab app.LabPathApp")

    version = version
    description = """
    Print the configured paths for the JupyterLab application

    The application path can be configured using the JUPYTERLAB_DIR environment variable.
    The user settings path can be configured using the JUPYTERLAB_SETTINGS_DIR
        environment variable or it will fall back to
        `/lab/user-settings` in the default Jupyter configuration directory.
    The workspaces path can be configured using the JUPYTERLAB_WORKSPACES_DIR
        environment variable or it will fall back to
        '/lab/workspaces' in the default Jupyter configuration directory.
    """

    def start(self):
        print("#######################################Lab app.LabPathApp")
        print('Application directory:   %s' % get_app_dir())
        print('User Settings directory: %s' % get_user_settings_dir())
        print('Workspaces directory %s' % get_workspaces_dir())


lab_aliases = dict(aliases)
lab_aliases['app-dir'] = 'LabApp.app_dir'

lab_flags = dict(flags)
lab_flags['core-mode'] = (
    {'LabApp': {'core_mode': True}},
    "Start the app in core mode."
)
lab_flags['dev-mode'] = (
    {'LabApp': {'dev_mode': True}},
    "Start the app in dev mode for running from source."
)
lab_flags['watch'] = (
    {'LabApp': {'watch': True}},
    "Start the app in watch mode."
)


class LabApp(NotebookApp):
    # def __init__(self):
    #    print("#######################################Lab app.LabApp")

    version = version

    description = """
    JupyterLab - An extensible computational environment for Jupyter.

    This launches a Tornado based HTML Server that serves up an
    HTML5/Javascript JupyterLab client.

    JupyterLab has three different modes of running:

    * Core mode (`--core-mode`): in this mode JupyterLab will run using the JavaScript
      assets contained in the installed `jupyterlab` Python package. In core mode, no
      extensions are enabled. This is the default in a stable JupyterLab release if you
      have no extensions installed.
    * Dev mode (`--dev-mode`): uses the unpublished local JavaScript packages in the
      `dev_mode` folder.  In this case JupyterLab will show a red stripe at the top of
      the page.  It can only be used if JupyterLab is installed as `pip install -e .`.
    * App mode: JupyterLab allows multiple JupyterLab "applications" to be
      created by the user with different combinations of extensions. The `--app-dir` can
      be used to set a directory for different applications. The default application
      path can be found using `jupyter lab path`.
    """

    examples = """
        jupyter lab                       # start JupyterLab
        jupyter lab --dev-mode            # start JupyterLab in development mode, with no extensions
        jupyter lab --core-mode           # start JupyterLab in core mode, with no extensions
        jupyter lab --app-dir=~/myjupyterlabapp # start JupyterLab with a particular set of extensions
        jupyter lab --certfile=mycert.pem # use SSL/TLS certificate
    """

    aliases = lab_aliases
    flags = lab_flags

    subcommands = dict(
        build=(LabBuildApp, LabBuildApp.description.splitlines()[0]),
        clean=(LabCleanApp, LabCleanApp.description.splitlines()[0]),
        path=(LabPathApp, LabPathApp.description.splitlines()[0]),
        paths=(LabPathApp, LabPathApp.description.splitlines()[0])
    )

    default_url = Unicode('/lab', config=True,
        help="The default URL to redirect to from `/`")

    app_dir = Unicode(get_app_dir(), config=True,
        help="The app directory to launch JupyterLab from.")

    user_settings_dir = Unicode(get_user_settings_dir(), config=True,
        help="The directory for user settings.")

    workspaces_dir = Unicode(get_workspaces_dir(), config=True,
        help="The directory for workspaces")

    core_mode = Bool(False, config=True,
        help="""Whether to start the app in core mode. In this mode, JupyterLab
        will run using the JavaScript assets that are within the installed
        JupyterLab Python package. In core mode, third party extensions are disabled.
        The `--dev-mode` flag is an alias to this to be used when the Python package
        itself is installed in development mode (`pip install -e .`).
        """)

    dev_mode = Bool(False, config=True,
        help="""Whether to start the app in dev mode. Uses the unpublished local
        JavaScript packages in the `dev_mode` folder.  In this case JupyterLab will
        show a red stripe at the top of the page.  It can only be used if JupyterLab
        is installed as `pip install -e .`.
        """)

    watch = Bool(False, config=True,
        help="Whether to serve the app in watch mode")

    def init_server_extensions(self):
        print('!!!!!!!!!!!!!!!!!!!!!!!!!!init_server_extensions')
        """Load any extensions specified by config.

        Import the module, then call the load_jupyter_server_extension function,
        if one exists.

        If the JupyterLab server extension is not enabled, it will
        be manually loaded with a warning.

        The extension API is experimental, and may change in future releases.
        """
        super(LabApp, self).init_server_extensions()
        msg = 'JupyterLab server extension not enabled, manually loading...'
        if not self.nbserver_extensions.get('jupyterlab', False):
            self.log.warn(msg)
            print('!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!log')
            load_jupyter_server_extension(self)

#------------------------------------------------------------------------
# Main entry point
#------------------------------------------------------------------------
main = launch_new_instance = LabApp.launch_instance

if __name__ == '__main__':
    print("#######################################Lab app")
    main()
```

```python
# coding: utf-8
# jlpmapp.py
"""A Jupyter-aware wrapper for the yarn package manager"""

# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.
import sys

import os
from jupyterlab_launcher.process import which, subprocess

HERE = os.path.dirname(os.path.abspath(__file__))
YARN_PATH = os.path.join(HERE, 'staging', 'yarn.js')


def execvp(cmd, argv):
    """Execvp, except on Windows where it uses Popen.

    The first argument, by convention, should point to the filename
    associated with the file being executed.

    Python provides execvp on Windows, but its behavior is problematic
    (Python bug#9148).
    """
    cmd = which(cmd)
    if os.name == 'nt':
        import signal
        import sys
        p = subprocess.Popen([cmd] + argv[1:])
        # Don't raise KeyboardInterrupt in the parent process.
        # Set this after spawning, to avoid subprocess inheriting handler.
        signal.signal(signal.SIGINT, signal.SIG_IGN)
        p.wait()
        sys.exit(p.returncode)
    else:
        os.execvp(cmd, argv)

def main(argv=None):
    """Run node and return the result.
    """
    # Make sure node is available.
    print('#########################jlpmapp.py ')
    argv = argv or sys.argv[1:]
    execvp('node', ['node', YARN_PATH] + argv)
```

```python
# coding: utf-8
# labextensions.py
"""Jupyter LabExtension Entry Points."""

# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.
from __future__ import print_function

import os
import sys
import traceback

from copy import copy

from jupyter_core.application import JupyterApp, base_flags, base_aliases

from traitlets import Bool, Unicode

from .commands import (
    install_extension, uninstall_extension, list_extensions,
    enable_extension, disable_extension, check_extension,
    link_package, unlink_package, build, get_app_version, HERE,
    update_extension,
)


flags = dict(base_flags)
flags['no-build'] = (
    {'BaseExtensionApp': {'should_build': False}},
    "Defer building the app after the action."
)
flags['dev-build'] = (
    {'BaseExtensionApp': {'dev_build': True}},
    "Build in Development mode"
)
flags['clean'] = (
    {'BaseExtensionApp': {'should_clean': True}},
    "Cleanup intermediate files after the action."
)

check_flags = copy(flags)
check_flags['installed'] = (
    {'CheckLabExtensionsApp': {'should_check_installed_only': True}},
    "Check only if the extension is installed."
)

update_flags = copy(flags)
update_flags['all'] = (
    {'UpdateLabExtensionApp': {'all': True}},
    "Update all extensions"
)

aliases = dict(base_aliases)
aliases['app-dir'] = 'BaseExtensionApp.app_dir'

VERSION = get_app_version()


class BaseExtensionApp(JupyterApp):
    version = VERSION
    flags = flags
    aliases = aliases

    app_dir = Unicode('', config=True,
        help="The app directory to target")

    should_build = Bool(True, config=True,
        help="Whether to build the app after the action")

    dev_build = Bool(False, config=True,
        help="Whether to build in dev mode (defaults to production mode)")

    should_clean = Bool(False, config=True,
        help="Whether temporary files should be cleaned up after building jupyterlab")

    def start(self):
        if self.app_dir and self.app_dir.startswith(HERE):
            raise ValueError('Cannot run lab extension commands in core app')
        try:
            ans = self.run_task()
            if ans and self.should_build:
                command = 'build:prod' if not self.dev_build else 'build'
                build(app_dir=self.app_dir, clean_staging=self.should_clean,
                      logger=self.log, command=command)
        except Exception as ex:
            _, _, exc_traceback = sys.exc_info()
            msg = traceback.format_exception(ex.__class__, ex, exc_traceback)
            for line in msg:
                self.log.debug(line)
            self.log.error('\nErrored, use --debug for full output:')
            self.log.error(msg[-1].strip())
            sys.exit(1)

    def run_task(self):
        pass

    def _log_format_default(self):
        """A default format for messages"""
        return "%(message)s"


class InstallLabExtensionApp(BaseExtensionApp):
    description = "Install labextension(s)"

    def run_task(self):
        self.extra_args = self.extra_args or [os.getcwd()]
        return any([
            install_extension(arg, self.app_dir, logger=self.log)
            for arg in self.extra_args
        ])


class UpdateLabExtensionApp(BaseExtensionApp):
    description = "Update labextension(s)"
    flags = update_flags

    all = Bool(False, config=True,
        help="Whether to update all extensions")

    def run_task(self):
        if not self.all and not self.extra_args:
            self.log.warn('Specify an extension to update, or use --all to update all extensions')
            return False
        if self.all:
            return update_extension(all_=True, app_dir=self.app_dir, logger=self.log)
        return any([
            update_extension(name=arg, app_dir=self.app_dir, logger=self.log)
            for arg in self.extra_args
        ])


class LinkLabExtensionApp(BaseExtensionApp):
    description = """
    Link local npm packages that are not lab extensions.

    Links a package to the JupyterLab build process. A linked
    package is manually re-installed from its source location when
    `jupyter lab build` is run.
    """
    should_build = Bool(True, config=True,
        help="Whether to build the app after the action")

    def run_task(self):
        self.extra_args = self.extra_args or [os.getcwd()]
        return any([
            link_package(arg, self.app_dir, logger=self.log)
            for arg in self.extra_args
        ])


class UnlinkLabExtensionApp(BaseExtensionApp):
    description = "Unlink packages by name or path"

    def run_task(self):
        self.extra_args = self.extra_args or [os.getcwd()]
        return any([
            unlink_package(arg, self.app_dir, logger=self.log)
            for arg in self.extra_args
        ])


class UninstallLabExtensionApp(BaseExtensionApp):
    description = "Uninstall labextension(s) by name"

    def run_task(self):
        self.extra_args = self.extra_args or [os.getcwd()]
        return any([
            uninstall_extension(arg, self.app_dir, logger=self.log)
            for arg in self.extra_args
        ])


class ListLabExtensionsApp(BaseExtensionApp):
    description = "List the installed labextensions"

    def run_task(self):
        list_extensions(self.app_dir, logger=self.log)


class EnableLabExtensionsApp(BaseExtensionApp):
    description = "Enable labextension(s) by name"

    def run_task(self):
        [enable_extension(arg, self.app_dir, logger=self.log)
         for arg in self.extra_args]


class DisableLabExtensionsApp(BaseExtensionApp):
    description = "Disable labextension(s) by name"

    def run_task(self):
        [disable_extension(arg, self.app_dir, logger=self.log)
         for arg in self.extra_args]


class CheckLabExtensionsApp(BaseExtensionApp):
    description = "Check labextension(s) by name"
    flags = check_flags

    should_check_installed_only = Bool(False, config=True,
        help="Whether it should check only if the extensions is installed")

    def run_task(self):
        all_enabled = all(
            check_extension(
                arg, self.app_dir,
                self.should_check_installed_only,
                logger=self.log)
            for arg in self.extra_args)
        if not all_enabled:
            exit(1)


_examples = """
jupyter labextension list                        # list all configured labextensions
jupyter labextension install <extension name>    # install a labextension
jupyter labextension uninstall <extension name>  # uninstall a labextension
"""


class LabExtensionApp(JupyterApp):
    """Base jupyter labextension command entry point"""
    name = "jupyter labextension"
    version = VERSION
    description = "Work with JupyterLab extensions"
    examples = _examples

    subcommands = dict(
        install=(InstallLabExtensionApp, "Install labextension(s)"),
        update=(UpdateLabExtensionApp, "Update labextension(s)"),
        uninstall=(UninstallLabExtensionApp, "Uninstall labextension(s)"),
        list=(ListLabExtensionsApp, "List labextensions"),
        link=(LinkLabExtensionApp, "Link labextension(s)"),
        unlink=(UnlinkLabExtensionApp, "Unlink labextension(s)"),
        enable=(EnableLabExtensionsApp, "Enable labextension(s)"),
        disable=(DisableLabExtensionsApp, "Disable labextension(s)"),
        check=(CheckLabExtensionsApp, "Check labextension(s)"),
    )

    def start(self):
        """Perform the App's functions as configured"""
        super(LabExtensionApp, self).start()

        # The above should have called a subcommand and raised NoStart; if we
        # get here, it didn't, so we should self.log.info a message.
        subcmds = ", ".join(sorted(self.subcommands))
        sys.exit("Please supply at least one subcommand: %s" % subcmds)


main = LabExtensionApp.launch_instance

if __name__ == '__main__':
    print("!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!labextensions.py")
    sys.exit(main())
```

```python
# labhubapp
import os
from .labapp import LabApp

try:
    from jupyterhub.singleuser import SingleUserNotebookApp
except ImportError:
    SingleUserLabApp = None
    raise ImportError('You must have jupyterhub installed for this to work.')
else:
    class SingleUserLabApp(SingleUserNotebookApp, LabApp):

        def init_webapp(self, *args, **kwargs):
            super().init_webapp(*args, **kwargs)
            settings = self.web_app.settings
            if 'page_config_data' not in settings:
                settings['page_config_data'] = {}
            settings['page_config_data']['hub_prefix'] = self.hub_prefix
            settings['page_config_data']['hub_host'] = self.hub_host
            settings['page_config_data']['hub_user'] = self.user
            api_token = os.getenv('JUPYTERHUB_API_TOKEN')
            if not api_token:
                api_token = ''
            if not self.token:
                try:
                    self.token = api_token
                except AttributeError:
                    self.log.error("Can't set self.token")
            settings['page_config_data']['token'] = api_token


def main(argv=None):
    return SingleUserLabApp.launch_instance(argv)

if __name__ == "__main__":
    print("!!!!!!!!!!!!!!!!!!!!!!!!!!!!!labhubapp.py ")
    main()
```

```python
# -*- coding: utf-8 -*-
# selenium_check.py
from __future__ import print_function, absolute_import

from concurrent.futures import ThreadPoolExecutor
from hashlib import sha256
import json
import os
import platform
import sys
import tarfile
import time
import zipfile

from tornado.httpclient import HTTPClient
from tornado.ioloop import IOLoop
from notebook.notebookapp import flags, aliases
from traitlets import Bool, Unicode

from selenium import webdriver
from .labapp import LabApp


here = os.path.dirname(__file__)
GECKO_PATH = os.path.join(here, 'geckodriver')
GECKO_VERSION = '0.19.1'

# Note: These were obtained by downloading the files from
# https://github.com/mozilla/geckodriver/releases
# and computing the sha using `shasum -a 256`
GECKO_SHA = dict(
    Linux='7f55c4c89695fd1e6f8fc7372345acc1e2dbaa4a8003cee4bd282eed88145937',
    Darwin='d914e96aa88d5950c65aa2b5d6ca0976e15bbbe20d788dde3bf3906b633bd675',
    Windows='b1c180842aa127686b93b4bf8570790c26a13dcb4c703a073404e0918de42090'
)
GECKO_TAR_NAME = dict(
    Linux='linux64.tar.gz',
    Darwin='macos.tar.gz',
    Windows='win64.zip'
)


def ensure_geckodriver(log):
    print('ensure_geckodriver!!!!!!!!!!!!!!!!!!!!!!!!')
    """Ensure a local copy of the geckodriver.
    """
    # Check for existing geckodriver file.
    if os.path.exists(GECKO_PATH):
        return True

    system = platform.system()
    if system not in GECKO_SHA:
        log.error('Unsupported platform %s' % system)
        return

    sha = GECKO_SHA[system]
    name = GECKO_TAR_NAME[system]

    url = ('https://github.com/mozilla/geckodriver/releases/'
           'download/v%s/geckodriver-v%s-%s'
           % (GECKO_VERSION, GECKO_VERSION, name))

    log.info('Downloading geckodriver v(%s) from: %s' % (GECKO_VERSION, url))

    response = HTTPClient().fetch(url)

    log.info('Validating geckodriver...')

    if sha256(response.body).hexdigest() != sha:
        log.error('Downloaded geckodriver doesn\'t match expected:'
                  '\n\t%s !=\t%s',
                  sha256(response.body).hexdigest(),
                  sha)
        return False

    log.info('Writing %s...', GECKO_PATH)

    if system == 'Windows':
        fid = zipfile.ZipFile(response.buffer)
    else:
        fid = tarfile.open(mode='r|gz', fileobj=response.buffer)
    fid.extractall(here)
    fid.close()

    return True


test_flags = dict(flags)
test_flags['core-mode'] = (
    {'SeleniumApp': {'core_mode': True}},
    "Start the app in core mode."
)
test_flags['dev-mode'] = (
    {'SeleniumApp': {'dev_mode': True}},
    "Start the app in dev mode."
)


test_aliases = dict(aliases)
test_aliases['app-dir'] = 'SeleniumApp.app_dir'


class SeleniumApp(LabApp):
    # def __init__(self):
    #     print("SeleniumApp #############################")
    open_browser = Bool(False)
    base_url = '/foo/'
    ip = '127.0.0.1'
    flags = test_flags
    aliases = test_aliases

    def start(self):
        print("seleniumApp start !!!!!!!!!!!!1")
        web_app = self.web_app
        web_app.settings.setdefault('page_config_data', dict())
        web_app.settings['page_config_data']['seleniumTest'] = True
        web_app.settings['page_config_data']['buildAvailable'] = False

        pool = ThreadPoolExecutor()
        future = pool.submit(run_selenium, self.display_url, self.log)
        IOLoop.current().add_future(future, self._selenium_finished)
        super(SeleniumApp, self).start()

    def _selenium_finished(self, future):
        try:
            sys.exit(future.result())
        except Exception as e:
            self.log.error(str(e))
            sys.exit(1)


def run_selenium(url, log):
    """Run the selenium test and return an exit code.
    """
    if not ensure_geckodriver(log):
        return 1

    log.info('Starting Firefox Driver')
    executable = GECKO_PATH
    if os.name == 'nt':
        executable += '.exe'
    driver = webdriver.Firefox(executable_path=executable)

    log.info('Navigating to page: %s' % url)
    driver.get(url)

    # Start a poll loop.
    log.info('Waiting for application to start...')
    t0 = time.time()
    el = None
    while time.time() - t0 < 20:
        try:
            el = driver.find_element_by_id('seleniumResult')
            if el:
                break
        except Exception as e:
            pass

        # Avoid hogging the main thread.
        time.sleep(0.5)

    if not el:
        driver.quit()
        log.error('Application did not start properly')
        return 1

    errors = json.loads(el.get_attribute('textContent'))
    driver.quit()

    if os.path.exists('./geckodriver.log'):
            os.remove('./geckodriver.log')

    if errors:
        for error in errors:
            log.error(str(error))
        return 1

    log.info('Selenium test complete')
    return 0


if __name__ == '__main__':
    print("!!!!!!!!!!!!!!!!!!!!!!!!!!!!selenium_check.py")
    SeleniumApp.launch_instance()
```

```python
# semver.py
# -*- coding:utf-8 -*-
# This file comes from https://github.com/podhmo/python-semver/blob/b42e9896e391e086b773fc621b23fa299d16b874/semver/__init__.py
# 
# It is licensed under the following license:
# 
# MIT License

# Copyright (c) 2016 podhmo

# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:

# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.

# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

import logging
import re
logger = logging.getLogger(__name__)


SEMVER_SPEC_VERSION = '2.0.0'

# Python 2/3 compatibility
try:
  string_type = basestring
except NameError:
  string_type = str

class _R(object):
    def __init__(self, i):
        self.i = i

    def __call__(self):
        v = self.i
        self.i += 1
        return v

    def value(self):
        return self.i


class Extendlist(list):
    def __setitem__(self, i, v):
        try:
            list.__setitem__(self, i, v)
        except IndexError:
            if len(self) == i:
                self.append(v)
            else:
                raise


def list_get(xs, i):
    try:
        return xs[i]
    except IndexError:
        return None


R = _R(0)
src = Extendlist()
regexp = {}

# The following Regular Expressions can be used for tokenizing,
# validating, and parsing SemVer version strings.

# ## Numeric Identifier
# A single `0`, or a non-zero digit followed by zero or more digits.
# 数字标识符
NUMERICIDENTIFIER = R()
src[NUMERICIDENTIFIER] = '0|[1-9]\\d*'

NUMERICIDENTIFIERLOOSE = R()
src[NUMERICIDENTIFIERLOOSE] = '[0-9]+'


# ## Non-numeric Identifier
# Zero or more digits, followed by a letter or hyphen, and then zero or
# more letters, digits, or hyphens.

NONNUMERICIDENTIFIER = R()
src[NONNUMERICIDENTIFIER] = '\\d*[a-zA-Z-][a-zA-Z0-9-]*'

# ## Main Version
# Three dot-separated numeric identifiers.

MAINVERSION = R()
src[MAINVERSION] = ('(' + src[NUMERICIDENTIFIER] + ')\\.' +
                    '(' + src[NUMERICIDENTIFIER] + ')\\.' +
                    '(' + src[NUMERICIDENTIFIER] + ')')

MAINVERSIONLOOSE = R()
src[MAINVERSIONLOOSE] = ('(' + src[NUMERICIDENTIFIERLOOSE] + ')\\.' +
                         '(' + src[NUMERICIDENTIFIERLOOSE] + ')\\.' +
                         '(' + src[NUMERICIDENTIFIERLOOSE] + ')')


# ## Pre-release Version Identifier
# A numeric identifier, or a non-numeric identifier.

PRERELEASEIDENTIFIER = R()
src[PRERELEASEIDENTIFIER] = ('(?:' + src[NUMERICIDENTIFIER] +
                             '|' + src[NONNUMERICIDENTIFIER] + ')')

PRERELEASEIDENTIFIERLOOSE = R()
src[PRERELEASEIDENTIFIERLOOSE] = ('(?:' + src[NUMERICIDENTIFIERLOOSE] +
                                  '|' + src[NONNUMERICIDENTIFIER] + ')')


# ## Pre-release Version
# Hyphen, followed by one or more dot-separated pre-release version
# identifiers.

PRERELEASE = R()
src[PRERELEASE] = ('(?:-(' + src[PRERELEASEIDENTIFIER] +
                   '(?:\\.' + src[PRERELEASEIDENTIFIER] + ')*))')

PRERELEASELOOSE = R()
src[PRERELEASELOOSE] = ('(?:-?(' + src[PRERELEASEIDENTIFIERLOOSE] +
                        '(?:\\.' + src[PRERELEASEIDENTIFIERLOOSE] + ')*))')

# ## Build Metadata Identifier
# Any combination of digits, letters, or hyphens.

BUILDIDENTIFIER = R()
src[BUILDIDENTIFIER] = '[0-9A-Za-z-]+'

# ## Build Metadata
# Plus sign, followed by one or more period-separated build metadata
# identifiers.

BUILD = R()
src[BUILD] = ('(?:\\+(' + src[BUILDIDENTIFIER] +
              '(?:\\.' + src[BUILDIDENTIFIER] + ')*))')

#  ## Full Version String
#  A main version, followed optionally by a pre-release version and
#  build metadata.

#  Note that the only major, minor, patch, and pre-release sections of
#  the version string are capturing groups.  The build metadata is not a
#  capturing group, because it should not ever be used in version
#  comparison.

FULL = R()
FULLPLAIN = ('v?' + src[MAINVERSION] + src[PRERELEASE] + '?' + src[BUILD] + '?')

src[FULL] = '^' + FULLPLAIN + '$'

#  like full, but allows v1.2.3 and =1.2.3, which people do sometimes.
#  also, 1.0.0alpha1 (prerelease without the hyphen) which is pretty
#  common in the npm registry.
LOOSEPLAIN = ('[v=\\s]*' + src[MAINVERSIONLOOSE] +
              src[PRERELEASELOOSE] + '?' +
              src[BUILD] + '?')

LOOSE = R()
src[LOOSE] = '^' + LOOSEPLAIN + '$'

GTLT = R()
src[GTLT] = '((?:<|>)?=?)'

#  Something like "2.*" or "1.2.x".
#  Note that "x.x" is a valid xRange identifier, meaning "any version"
#  Only the first item is strictly required.
XRANGEIDENTIFIERLOOSE = R()
src[XRANGEIDENTIFIERLOOSE] = src[NUMERICIDENTIFIERLOOSE] + '|x|X|\\*'
XRANGEIDENTIFIER = R()
src[XRANGEIDENTIFIER] = src[NUMERICIDENTIFIER] + '|x|X|\\*'

XRANGEPLAIN = R()
src[XRANGEPLAIN] = ('[v=\\s]*(' + src[XRANGEIDENTIFIER] + ')' +
                    '(?:\\.(' + src[XRANGEIDENTIFIER] + ')' +
                    '(?:\\.(' + src[XRANGEIDENTIFIER] + ')' +
                    '(?:' + src[PRERELEASE] + ')?' +
                    src[BUILD] + '?' +
                    ')?)?')

XRANGEPLAINLOOSE = R()
src[XRANGEPLAINLOOSE] = ('[v=\\s]*(' + src[XRANGEIDENTIFIERLOOSE] + ')' +
                         '(?:\\.(' + src[XRANGEIDENTIFIERLOOSE] + ')' +
                         '(?:\\.(' + src[XRANGEIDENTIFIERLOOSE] + ')' +
                         '(?:' + src[PRERELEASELOOSE] + ')?' +
                         src[BUILD] + '?' +
                         ')?)?')

XRANGE = R()
src[XRANGE] = '^' + src[GTLT] + '\\s*' + src[XRANGEPLAIN] + '$'
XRANGELOOSE = R()
src[XRANGELOOSE] = '^' + src[GTLT] + '\\s*' + src[XRANGEPLAINLOOSE] + '$'

#  Tilde ranges.
#  Meaning is "reasonably at or greater than"
LONETILDE = R()
src[LONETILDE] = '(?:~>?)'

TILDETRIM = R()
src[TILDETRIM] = '(\\s*)' + src[LONETILDE] + '\\s+'
regexp[TILDETRIM] = re.compile(src[TILDETRIM], re.M)
tildeTrimReplace = r'\1~'

TILDE = R()
src[TILDE] = '^' + src[LONETILDE] + src[XRANGEPLAIN] + '$'
TILDELOOSE = R()
src[TILDELOOSE] = ('^' + src[LONETILDE] + src[XRANGEPLAINLOOSE] + '$')

#  Caret ranges.
#  Meaning is "at least and backwards compatible with"
LONECARET = R()
src[LONECARET] = '(?:\\^)'

CARETTRIM = R()
src[CARETTRIM] = '(\\s*)' + src[LONECARET] + '\\s+'
regexp[CARETTRIM] = re.compile(src[CARETTRIM], re.M)
caretTrimReplace = r'\1^'

CARET = R()
src[CARET] = '^' + src[LONECARET] + src[XRANGEPLAIN] + '$'
CARETLOOSE = R()
src[CARETLOOSE] = '^' + src[LONECARET] + src[XRANGEPLAINLOOSE] + '$'

#  A simple gt/lt/eq thing, or just "" to indicate "any version"
COMPARATORLOOSE = R()
src[COMPARATORLOOSE] = '^' + src[GTLT] + '\\s*(' + LOOSEPLAIN + ')$|^$'
COMPARATOR = R()
src[COMPARATOR] = '^' + src[GTLT] + '\\s*(' + FULLPLAIN + ')$|^$'


#  An expression to strip any whitespace between the gtlt and the thing
#  it modifies, so that `> 1.2.3` ==> `>1.2.3`
COMPARATORTRIM = R()
src[COMPARATORTRIM] = ('(\\s*)' + src[GTLT] +
                       '\\s*(' + LOOSEPLAIN + '|' + src[XRANGEPLAIN] + ')')

#  this one has to use the /g flag
regexp[COMPARATORTRIM] = re.compile(src[COMPARATORTRIM], re.M)
comparatorTrimReplace = r'\1\2\3'


#  Something like `1.2.3 - 1.2.4`
#  Note that these all use the loose form, because they'll be
#  checked against either the strict or loose comparator form
#  later.
HYPHENRANGE = R()
src[HYPHENRANGE] = ('^\\s*(' + src[XRANGEPLAIN] + ')' +
                    '\\s+-\\s+' +
                    '(' + src[XRANGEPLAIN] + ')' +
                    '\\s*$')

HYPHENRANGELOOSE = R()
src[HYPHENRANGELOOSE] = ('^\\s*(' + src[XRANGEPLAINLOOSE] + ')' +
                         '\\s+-\\s+' +
                         '(' + src[XRANGEPLAINLOOSE] + ')' +
                         '\\s*$')

#  Star ranges basically just allow anything at all.
STAR = R()
src[STAR] = '(<|>)?=?\\s*\\*'

# version name recovery for convinient
RECOVERYVERSIONNAME = R()
src[RECOVERYVERSIONNAME] = ('v?({n})(?:\\.({n}))?{pre}?'.format(n=src[NUMERICIDENTIFIER], pre=src[PRERELEASELOOSE]))

#  Compile to actual regexp objects.
#  All are flag-free, unless they were created above with a flag.
for i in range(R.value()):
    logger.debug("genregxp %s %s", i, src[i])
    if i not in regexp:
        regexp[i] = re.compile(src[i])


def parse(version, loose):
    if loose:
        r = regexp[LOOSE]
    else:
        r = regexp[FULL]
    m = r.search(version)
    if m:
        return semver(version, loose)
    else:
        return None


def valid(version, loose):
    v = parse(version, loose)
    if v.version:
        return v
    else:
        return None


def clean(version, loose):
    s = parse(version, loose)
    if s:
        return s.version
    else:
        return None


NUMERIC = re.compile("^\d+$")


def semver(version, loose):
    print('######################################semver.py --->>semver')
    if isinstance(version, SemVer):
        if version.loose == loose:
            return version
        else:
            version = version.version
    elif not isinstance(version, string_type):  # xxx:
        raise ValueError("Invalid Version: {}".format(version))

    """
    if (!(this instanceof SemVer))
       return new SemVer(version, loose);
    """
    return SemVer(version, loose)


make_semver = semver


class SemVer(object):
    def __init__(self, version, loose):
        print('######################################semver.py --->>SemVer')
        logger.debug("SemVer %s, %s", version, loose)
        self.loose = loose
        self.raw = version

        m = regexp[LOOSE if loose else FULL].search(version.strip())
        if not m:
            if not loose:
                raise ValueError("Invalid Version: {}".format(version))
            m = regexp[RECOVERYVERSIONNAME].search(version.strip())
            self.major = int(m.group(1)) if m.group(1) else 0
            self.minor = int(m.group(2)) if m.group(2) else 0
            self.patch = 0
            if not m.group(3):
                self.prerelease = []
            else:
                self.prerelease = [(int(id) if NUMERIC.search(id) else id)
                                   for id in m.group(3).split(".")]
        else:
            #  these are actually numbers
            self.major = int(m.group(1))
            self.minor = int(m.group(2))
            self.patch = int(m.group(3))
            #  numberify any prerelease numeric ids
            if not m.group(4):
                self.prerelease = []
            else:

                self.prerelease = [(int(id) if NUMERIC.search(id) else id)
                                   for id in m.group(4).split(".")]
            if m.group(5):
                self.build = m.group(5).split(".")
            else:
                self.build = []

        self.format()  # xxx:

    def format(self):
        self.version = "{}.{}.{}".format(self.major, self.minor, self.patch)
        if len(self.prerelease) > 0:
            self.version += ("-{}".format(".".join(str(v) for v in self.prerelease)))
        return self.version

    def __repr__(self):
        return "<SemVer {} >".format(self)

    def __str__(self):
        return self.version

    def compare(self, other):
        logger.debug('SemVer.compare %s %s %s', self.version, self.loose, other)
        if not isinstance(other, SemVer):
            other = make_semver(other, self.loose)
        result = self.compare_main(other) or self.compare_pre(other)
        logger.debug("compare result %s", result)
        return result

    def compare_main(self, other):
        if not isinstance(other, SemVer):
            other = make_semver(other, self.loose)

        return (compare_identifiers(str(self.major), str(other.major)) or
                compare_identifiers(str(self.minor), str(other.minor)) or
                compare_identifiers(str(self.patch), str(other.patch)))

    def compare_pre(self, other):
        if not isinstance(other, SemVer):
            other = make_semver(other, self.loose)

        #  NOT having a prerelease is > having one
        is_self_more_than_zero = len(self.prerelease) > 0
        is_other_more_than_zero = len(other.prerelease) > 0

        if not is_self_more_than_zero and is_other_more_than_zero:
            return 1
        elif is_self_more_than_zero and not is_other_more_than_zero:
            return -1
        elif not is_self_more_than_zero and not is_other_more_than_zero:
            return 0

        i = 0
        while True:
            a = list_get(self.prerelease, i)
            b = list_get(other.prerelease, i)
            logger.debug("prerelease compare %s: %s %s", i, a, b)
            i += 1
            if a is None and b is None:
                return 0
            elif b is None:
                return 1
            elif a is None:
                return -1
            elif a == b:
                continue
            else:
                return compare_identifiers(str(a), str(b))

    def inc(self, release, identifier=None):
        logger.debug("inc release %s %s", self.prerelease, release)
        if release == 'premajor':
            self.prerelease = []
            self.patch = 0
            self.minor = 0
            self.major += 1
            self.inc('pre', identifier=identifier)
        elif release == "preminor":
            self.prerelease = []
            self.patch = 0
            self.minor += 1
            self.inc('pre', identifier=identifier)
        elif release == "prepatch":
            # If this is already a prerelease, it will bump to the next version
            # drop any prereleases that might already exist, since they are not
            # relevant at this point.
            self.prerelease = []
            self.inc('patch', identifier=identifier)
            self.inc('pre', identifier=identifier)
        elif release == 'prerelease':
            # If the input is a non-prerelease version, this acts the same as
            # prepatch.
            if len(self.prerelease) == 0:
                self.inc("patch", identifier=identifier)
            self.inc("pre", identifier=identifier)
        elif release == "major":
            # If this is a pre-major version, bump up to the same major version.
            # Otherwise increment major.
            # 1.0.0-5 bumps to 1.0.0
            # 1.1.0 bumps to 2.0.0
            if self.minor != 0 or self.patch != 0 or len(self.prerelease) == 0:
                self.major += 1
            self.minor = 0
            self.patch = 0
            self.prerelease = []
        elif release == "minor":
            # If this is a pre-minor version, bump up to the same minor version.
            # Otherwise increment minor.
            # 1.2.0-5 bumps to 1.2.0
            # 1.2.1 bumps to 1.3.0
            if self.patch != 0 or len(self.prerelease) == 0:
                self.minor += 1
            self.patch = 0
            self.prerelease = []
        elif release == "patch":
            #  If this is not a pre-release version, it will increment the patch.
            #  If it is a pre-release it will bump up to the same patch version.
            #  1.2.0-5 patches to 1.2.0
            #  1.2.0 patches to 1.2.1
            if len(self.prerelease) == 0:
                self.patch += 1
            self.prerelease = []
        elif release == "pre":
            #  This probably shouldn't be used publically.
            #  1.0.0 "pre" would become 1.0.0-0 which is the wrong direction.
            logger.debug("inc prerelease %s", self.prerelease)
            if len(self.prerelease) == 0:
                self.prerelease = [0]
            else:
                i = len(self.prerelease) - 1
                while i >= 0:
                    if isinstance(self.prerelease[i], int):
                        self.prerelease[i] += 1
                        i -= 2
                    i -= 1
                # ## this is needless code in python ##
                # if i == -1:  # didn't increment anything
                #     self.prerelease.append(0)
            if identifier is not None:
                # 1.2.0-beta.1 bumps to 1.2.0-beta.2,
                # 1.2.0-beta.fooblz or 1.2.0-beta bumps to 1.2.0-beta.0
                if self.prerelease[0] == identifier:
                    if not isinstance(self.prerelease[1], int):
                        self.prerelease = [identifier, 0]
                else:
                    self.prerelease = [identifier, 0]
        else:
            raise ValueError('invalid increment argument: {}'.format(release))
        self.format()
        self.raw = self.version
        return self


def inc(version, release, loose, identifier=None):  # wow!
    try:
        return make_semver(version, loose).inc(release, identifier=identifier).version
    except Exception as e:
        logger.debug(e, exc_info=5)
        return None


def compare_identifiers(a, b):
    anum = NUMERIC.search(a)
    bnum = NUMERIC.search(b)

    if anum and bnum:
        a = int(a)
        b = int(b)

    if anum and not bnum:
        return -1
    elif bnum and not anum:
        return 1
    elif a < b:
        return -1
    elif a > b:
        return 1
    else:
        return 0


def rcompare_identifiers(a, b):
    return compare_identifiers(b, a)


def compare(a, b, loose):
    return make_semver(a, loose).compare(b)


def compare_loose(a, b):
    return compare(a, b, True)


def rcompare(a, b, loose):
    return compare(b, a, loose)


def make_key_function(loose):
    def key_function(version):
        v = make_semver(version, loose)
        key = (v.major, v.minor, v.patch)
        if v.prerelease:
            key = key + tuple(v.prerelease)
        else:
            #  NOT having a prerelease is > having one
            key = key + (float('inf'),)

        return key
    return key_function

loose_key_function = make_key_function(True)
full_key_function = make_key_function(True)


def sort(list, loose):
    keyf = loose_key_function if loose else full_key_function
    list.sort(key=keyf)
    return list


def rsort(list, loose):
    keyf = loose_key_function if loose else full_key_function
    list.sort(key=keyf, reverse=True)
    return list


def gt(a, b, loose):
    return compare(a, b, loose) > 0


def lt(a, b, loose):
    return compare(a, b, loose) < 0


def eq(a, b, loose):
    return compare(a, b, loose) == 0


def neq(a, b, loose):
    return compare(a, b, loose) != 0


def gte(a, b, loose):
    return compare(a, b, loose) >= 0


def lte(a, b, loose):
    return compare(a, b, loose) <= 0


def cmp(a, op, b, loose):
    logger.debug("cmp: %s", op)
    if op == "===":
        return a == b
    elif op == "!==":
        return a != b
    elif op == "" or op == "=" or op == "==":
        return eq(a, b, loose)
    elif op == "!=":
        return neq(a, b, loose)
    elif op == ">":
        return gt(a, b, loose)
    elif op == ">=":
        return gte(a, b, loose)
    elif op == "<":
        return lt(a, b, loose)
    elif op == "<=":
        return lte(a, b, loose)
    else:
        raise ValueError("Invalid operator: {}".format(op))


def comparator(comp, loose):
    if isinstance(comp, Comparator):
        if(comp.loose == loose):
            return comp
        else:
            comp = comp.value

    # if (!(this instanceof Comparator))
    #   return new Comparator(comp, loose)
    return Comparator(comp, loose)


make_comparator = comparator

ANY = object()


class Comparator(object):
    semver = None

    def __init__(self, comp, loose):
        logger.debug("comparator: %s %s", comp, loose)
        self.loose = loose
        self.parse(comp)

        if self.semver == ANY:
            self.value = ""
        else:
            self.value = self.operator + self.semver.version

    def parse(self, comp):
        if self.loose:
            r = regexp[COMPARATORLOOSE]
        else:
            r = regexp[COMPARATOR]
        logger.debug("parse comp=%s", comp)
        m = r.search(comp)

        if m is None:
            raise ValueError("Invalid comparator: {}".format(comp))

        self.operator = m.group(1)
        # if it literally is just '>' or '' then allow anything.
        if m.group(2) is None:
            self.semver = ANY
        else:
            self.semver = semver(m.group(2), self.loose)

    def __repr__(self):
        return '<SemVer Comparator "{}">'.format(self)

    def __str__(self):
        return self.value

    def test(self, version):
        logger.debug('Comparator, test %s, %s', version, self.loose)
        if self.semver == ANY:
            return True
        else:
            return cmp(version, self.operator, self.semver, self.loose)


def make_range(range_, loose):
    if isinstance(range_, Range) and range_.loose == loose:
        return range_

    # if (!(this instanceof Range))
    #    return new Range(range, loose);
    return Range(range_, loose)


class Range(object):
    def __init__(self, range_, loose):
        self.loose = loose
        #  First, split based on boolean or ||
        self.raw = range_
        xs = [self.parse_range(r.strip()) for r in re.split(r"\s*\|\|\s*", range_)]
        self.set = [r for r in xs if r]

        if not len(self.set):
            raise ValueError("Invalid SemVer Range: {}".format(range_))

        self.format()

    def __repr__(self):
        return '<SemVer Range "{}">'.format(self.range)

    def format(self):
        self.range = "||".join([" ".join(c.value for c in comps).strip() for comps in self.set]).strip()
        logger.debug("Range format %s", self.range)
        return self.range

    def __str__(self):
        return self.range

    def parse_range(self, range_):
        loose = self.loose
        logger.debug('range %s %s', range_, loose)
        #  `1.2.3 - 1.2.4` => `>=1.2.3 <=1.2.4`
        if loose:
            hr = regexp[HYPHENRANGELOOSE]
        else:
            hr = regexp[HYPHENRANGE]

        range_ = hr.sub(hyphen_replace, range_,)
        logger.debug('hyphen replace %s', range_)

        #  `> 1.2.3 < 1.2.5` => `>1.2.3 <1.2.5`
        range_ = regexp[COMPARATORTRIM].sub(comparatorTrimReplace, range_)
        logger.debug('comparator trim %s, %s', range_, regexp[COMPARATORTRIM])

        #  `~ 1.2.3` => `~1.2.3`
        range_ = regexp[TILDETRIM].sub(tildeTrimReplace, range_)

        #  `^ 1.2.3` => `^1.2.3`
        range_ = regexp[CARETTRIM].sub(caretTrimReplace, range_)

        #  normalize spaces
        range_ = " ".join(re.split("\s+", range_))

        #  At this point, the range is completely trimmed and
        #  ready to be split into comparators.
        if loose:
            comp_re = regexp[COMPARATORLOOSE]
        else:
            comp_re = regexp[COMPARATOR]
        set_ = re.split("\s+", ' '.join([parse_comparator(comp, loose) for comp in range_.split(" ")]))
        if self.loose:
            # in loose mode, throw out any that are not valid comparators
            set_ = [comp for comp in set_ if comp_re.search(comp)]
        set_ = [make_comparator(comp, loose) for comp in set_]
        return set_

    def test(self, version):
        if not version:  # xxx
            return False

        if isinstance(version, string_type):
            version = make_semver(version, loose=self.loose)

        for e in self.set:
            if test_set(e, version):
                return True
        return False


#  Mostly just for testing and legacy API reasons
def to_comparators(range_, loose):
    return [" ".join([c.value for c in comp]).strip().split(" ")
            for comp in make_range(range_, loose).set]


#  comprised of xranges, tildes, stars, and gtlt's at this point.
#  already replaced the hyphen ranges
#  turn into a set of JUST comparators.

def parse_comparator(comp, loose):
    logger.debug('comp %s', comp)
    comp = replace_carets(comp, loose)
    logger.debug('caret %s', comp)
    comp = replace_tildes(comp, loose)
    logger.debug('tildes %s', comp)
    comp = replace_xranges(comp, loose)
    logger.debug('xrange %s', comp)
    comp = replace_stars(comp, loose)
    logger.debug('stars %s', comp)
    return comp


def is_x(id):
    return id is None or id == "" or id.lower() == "x" or id == "*"


#  ~, ~> --> * (any, kinda silly)
#  ~2, ~2.x, ~2.x.x, ~>2, ~>2.x ~>2.x.x --> >=2.0.0 <3.0.0
#  ~2.0, ~2.0.x, ~>2.0, ~>2.0.x --> >=2.0.0 <2.1.0
#  ~1.2, ~1.2.x, ~>1.2, ~>1.2.x --> >=1.2.0 <1.3.0
#  ~1.2.3, ~>1.2.3 --> >=1.2.3 <1.3.0
#  ~1.2.0, ~>1.2.0 --> >=1.2.0 <1.3.0

def replace_tildes(comp, loose):
    return " ".join([replace_tilde(c, loose)
                     for c in re.split("\s+", comp.strip())])


def replace_tilde(comp, loose):
    if loose:
        r = regexp[TILDELOOSE]
    else:
        r = regexp[TILDE]

    def repl(mob):
        _ = mob.group(0)
        M, m, p, pr, _ = mob.groups()
        logger.debug("tilde %s %s %s %s %s %s", comp, _, M, m, p, pr)
        if is_x(M):
            ret = ""
        elif is_x(m):
            ret = '>=' + M + '.0.0 <' + str(int(M) + 1) + '.0.0'
        elif is_x(p):
            # ~1.2 == >=1.2.0 <1.3.0
            ret = '>=' + M + '.' + m + '.0 <' + M + '.' + str(int(m) + 1) + '.0'
        elif pr:
            logger.debug("replaceTilde pr %s", pr)
            if (pr[0] != "-"):
                pr = '-' + pr
            ret = '>=' + M + '.' + m + '.' + p + pr + ' <' + M + '.' + str(int(m) + 1) + '.0'
        else:
            #  ~1.2.3 == >=1.2.3 <1.3.0
            ret = '>=' + M + '.' + m + '.' + p + ' <' + M + '.' + str(int(m) + 1) + '.0'
        logger.debug('tilde return, %s', ret)
        return ret
    return r.sub(repl, comp)


#  ^ --> * (any, kinda silly)
#  ^2, ^2.x, ^2.x.x --> >=2.0.0 <3.0.0
#  ^2.0, ^2.0.x --> >=2.0.0 <3.0.0
#  ^1.2, ^1.2.x --> >=1.2.0 <2.0.0
#  ^1.2.3 --> >=1.2.3 <2.0.0
#  ^1.2.0 --> >=1.2.0 <2.0.0
def replace_carets(comp, loose):
    return " ".join([replace_caret(c, loose)
                     for c in re.split("\s+", comp.strip())])


def replace_caret(comp, loose):
    if loose:
        r = regexp[CARETLOOSE]
    else:
        r = regexp[CARET]

    def repl(mob):
        m0 = mob.group(0)
        M, m, p, pr, _ = mob.groups()
        logger.debug("caret %s %s %s %s %s %s", comp, m0, M, m, p, pr)

        if is_x(M):
            ret = ""
        elif is_x(m):
            ret = '>=' + M + '.0.0 <' + str((int(M) + 1)) + '.0.0'
        elif is_x(p):
            if M == "0":
                ret = '>=' + M + '.' + m + '.0 <' + M + '.' + str((int(m) + 1)) + '.0'
            else:
                ret = '>=' + M + '.' + m + '.0 <' + str(int(M) + 1) + '.0.0'
        elif pr:
            logger.debug('replaceCaret pr %s', pr)
            if pr[0] != "-":
                pr = "-" + pr
            if M == "0":
                if m == "0":
                    ret = '>=' + M + '.' + m + '.' + (p or "") + pr + ' <' + M + '.' + m + "." + str(int(p or 0) + 1)
                else:
                    ret = '>=' + M + '.' + m + '.' + (p or "") + pr + ' <' + M + '.' + str(int(m) + 1) + '.0'
            else:
                ret = '>=' + M + '.' + m + '.' + (p or "") + pr + ' <' + str(int(M) + 1) + '.0.0'
        else:
            if M == "0":
                if m == "0":
                    ret = '>=' + M + '.' + m + '.' + (p or "") + ' <' + M + '.' + m + "." + str(int(p or 0) + 1)
                else:
                    ret = '>=' + M + '.' + m + '.' + (p or "") + ' <' + M + '.' + str((int(m) + 1)) + '.0'
            else:
                ret = '>=' + M + '.' + m + '.' + (p or "") + ' <' + str(int(M) + 1) + '.0.0'
        logger.debug('caret return %s', ret)
        return ret

    return r.sub(repl, comp)


def replace_xranges(comp, loose):
    logger.debug('replaceXRanges %s %s', comp, loose)
    return " ".join([replace_xrange(c, loose)
                     for c in re.split("\s+", comp.strip())])


def replace_xrange(comp, loose):
    comp = comp.strip()
    if loose:
        r = regexp[XRANGELOOSE]
    else:
        r = regexp[XRANGE]

    def repl(mob):
        ret = mob.group(0)
        gtlt, M, m, p, pr, _ = mob.groups()

        logger.debug("xrange %s %s %s %s %s %s %s", comp, ret, gtlt, M, m, p, pr)

        xM = is_x(M)
        xm = xM or is_x(m)
        xp = xm or is_x(p)
        any_x = xp

        if gtlt == "=" and any_x:
            gtlt = ""

        logger.debug("xrange gtlt=%s any_x=%s", gtlt, any_x)
        if xM:
            if gtlt == '>' or gtlt == '<':
                # nothing is allowed
                ret = '<0.0.0'
            else:
                ret = '*'
        elif gtlt and any_x:
            # replace X with 0, and then append the -0 min-prerelease
            if xm:
                m = 0
            if xp:
                p = 0

            if gtlt == ">":
                #  >1 => >=2.0.0
                #  >1.2 => >=1.3.0
                #  >1.2.3 => >= 1.2.4
                gtlt = ">="
                if xm:
                    M = int(M) + 1
                    m = 0
                    p = 0
                elif xp:
                    m = int(m) + 1
                    p = 0
            elif gtlt == '<=':
                # <=0.7.x is actually <0.8.0, since any 0.7.x should
                # pass.  Similarly, <=7.x is actually <8.0.0, etc.
                gtlt = '<'
                if xm:
                    M = int(M) + 1
                else:
                    m = int(m) + 1

            ret = gtlt + str(M) + '.' + str(m) + '.' + str(p)
        elif xm:
            ret = '>=' + M + '.0.0 <' + str(int(M) + 1) + '.0.0'
        elif xp:
            ret = '>=' + M + '.' + m + '.0 <' + M + '.' + str(int(m) + 1) + '.0'
        logger.debug('xRange return %s', ret)

        return ret
    return r.sub(repl, comp)


#  Because * is AND-ed with everything else in the comparator,
#  and '' means "any version", just remove the *s entirely.
def replace_stars(comp, loose):
    logger.debug('replaceStars %s %s', comp, loose)
    #  Looseness is ignored here.  star is always as loose as it gets!
    return regexp[STAR].sub("", comp.strip())


#  This function is passed to string.replace(re[HYPHENRANGE])
#  M, m, patch, prerelease, build
#  1.2 - 3.4.5 => >=1.2.0 <=3.4.5
#  1.2.3 - 3.4 => >=1.2.0 <3.5.0 Any 3.4.x will do
#  1.2 - 3.4 => >=1.2.0 <3.5.0
def hyphen_replace(mob):
    from_, fM, fm, fp, fpr, fb, to, tM, tm, tp, tpr, tb = mob.groups()
    if is_x(fM):
        from_ = ""
    elif is_x(fm):
        from_ = '>=' + fM + '.0.0'
    elif is_x(fp):
        from_ = '>=' + fM + '.' + fm + '.0'
    else:
        from_ = ">=" + from_

    if is_x(tM):
        to = ""
    elif is_x(tm):
        to = '<' + str(int(tM) + 1) + '.0.0'
    elif is_x(tp):
        to = '<' + tM + '.' + str(int(tm) + 1) + '.0'
    elif tpr:
        to = '<=' + tM + '.' + tm + '.' + tp + '-' + tpr
    else:
        to = '<=' + to
    return (from_ + ' ' + to).strip()


def test_set(set_, version):
    for e in set_:
        if not e.test(version):
            return False
    if len(version.prerelease) > 0:
        # Find the set of versions that are allowed to have prereleases
        # For example, ^1.2.3-pr.1 desugars to >=1.2.3-pr.1 <2.0.0
        # That should allow `1.2.3-pr.2` to pass.
        # However, `1.2.4-alpha.notready` should NOT be allowed,
        # even though it's within the range set by the comparators.
        for e in set_:
            if e.semver == ANY:
                continue
            if len(e.semver.prerelease) > 0:
                allowed = e.semver
                if allowed.major == version.major and allowed.minor == version.minor and allowed.patch == version.patch:
                    return True
        # Version has a -pre, but it's not one of the ones we like.
        return False
    return True


def satisfies(version, range_, loose=False):
    try:
        range_ = make_range(range_, loose)
    except Exception as e:
        return False
    return range_.test(version)


def max_satisfying(versions, range_, loose=False):
    try:
        range_ob = make_range(range_, loose=loose)
    except:
        return None
    max_ = None
    max_sv = None
    for v in versions:
        if range_ob.test(v):  # satisfies(v, range_, loose=loose)
            if max_ is None or max_sv.compare(v) == -1:  # compare(max, v, true)
                max_ = v
                max_sv = make_semver(max_, loose=loose)
    return max_


def valid_range(range_, loose):
    try:
        #  Return '*' instead of '' so that truthiness works.
        #  This will throw if it's invalid anyway
        return make_range(range_, loose).range or "*"
    except:
        return None


#  Determine if version is less than all the versions possible in the range
def ltr(version, range_, loose):
    return outside(version, range_, "<", loose)


#  Determine if version is greater than all the versions possible in the range.
def rtr(version, range_, loose):
    return outside(version, range_, ">", loose)


def outside(version, range_, hilo, loose):
    version = make_semver(version, loose)
    range_ = make_range(range_, loose)

    if hilo == ">":
        gtfn = gt
        ltefn = lte
        ltfn = lt
        comp = ">"
        ecomp = ">="
    elif hilo == "<":
        gtfn = lt
        ltefn = gte
        ltfn = gt
        comp = "<"
        ecomp = "<="
    else:
        raise ValueError("Must provide a hilo val of '<' or '>'")

    #  If it satisifes the range it is not outside
    if satisfies(version, range_, loose):
        return False

    #  From now on, variable terms are as if we're in "gtr" mode.
    #  but note that everything is flipped for the "ltr" function.
    for comparators in range_.set:
        high = None
        low = None

        for comparator in comparators:
            high = high or comparator
            low = low or comparator

            if gtfn(comparator.semver, high.semver, loose):
                high = comparator
            elif ltfn(comparator.semver, low.semver, loose):
                low = comparator

    #  If the edge version comparator has a operator then our version
    #  isn't outside it
    if high.operator == comp or high.operator == ecomp:
        return False

    #  If the lowest version comparator has an operator and our version
    #  is less than it then it isn't higher than the range
    if (not low.operator or low.operator == comp) and ltefn(version, low.semver):
        return False
    elif low.operator == ecomp and ltfn(version, low.semver):
        return False
    return True

```

```python
# update.py
# coding: utf-8
# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.
from os.path import join as pjoin
import json
import os
import shutil

HERE = os.path.dirname(__file__)
print("!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!update.py")

# Get the dev mode package.json file.
dev_path = os.path.realpath(pjoin(HERE, '..', 'dev_mode'))
with open(pjoin(dev_path, 'package.json')) as fid:
    data = json.load(fid)


# Update the values that need to change and write to staging.
data['scripts']['build'] = 'webpack'
data['scripts']['watch'] = 'webpack --watch'
data['scripts']['build:prod'] = (
    "webpack --define process.env.NODE_ENV=\"'production'\"")
data['jupyterlab']['outputDir'] = '..'
data['jupyterlab']['staticDir'] = '../static'
data['jupyterlab']['linkedPackages'] = dict()

staging = pjoin(HERE, 'staging')

with open(pjoin(staging, 'package.json'), 'w') as fid:
    json.dump(data, fid)

# Update our index file and webpack file.
for fname in ['index.js', 'webpack.config.js']:
    shutil.copy(pjoin(dev_path, fname), pjoin(staging, fname))


# Get a new yarn lock file.

target = os.path.join(app_dir, 'staging', 'yarn.lock')
shutil.copy(target, os.path.join(staging, 'yarn.lock'))

```

```python
# extension.py
# coding: utf-8
"""A tornado based Jupyter lab server."""

# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.

# ----------------------------------------------------------------------------
# Module globals
# ----------------------------------------------------------------------------
import os

DEV_NOTE = """You're running JupyterLab from source.
If you're working on the TypeScript sources of JupyterLab, try running

    jupyter lab --dev-mode --watch


to have the system incrementally watch and build JupyterLab for you, as you
make changes.
"""


CORE_NOTE = """
Running the core application with no additional extensions or settings
"""


def load_jupyter_server_extension(nbapp):

    """Load the JupyterLab server extension.
    """
    # Delay imports to speed up jlpmapp
    from json import dumps
    from jupyterlab_launcher import add_handlers, LabConfig
    from notebook.utils import url_path_join as ujoin, url_escape
    from notebook._version import version_info
    from tornado.ioloop import IOLoop
    from markupsafe import Markup
    from .build_handler import build_path, Builder, BuildHandler
    from .exte.Type_handler import Models_Handler, m_build_path
    from .commands import (
        get_app_dir, get_user_settings_dir, watch, ensure_dev, watch_dev,
        pjoin, DEV_DIR, HERE, get_app_info, ensure_core, get_workspaces_dir
    )

    web_app = nbapp.web_app
    logger = nbapp.log
    config = LabConfig()
    app_dir = getattr(nbapp, 'app_dir', get_app_dir())
    user_settings_dir = getattr(
        nbapp, 'user_settings_dir', get_user_settings_dir()
    )
    workspaces_dir = getattr(
        nbapp, 'workspaces_dir', get_workspaces_dir()
    )

    # Print messages.
    logger.info('JupyterLab beta preview extension loaded from %s' % HERE)
    logger.info('JupyterLab application directory is %s' % app_dir)

    config.app_name = 'JupyterLab Beta'
    config.app_namespace = 'jupyterlab'
    config.page_url = '/lab'
    config.cache_files = True

    # Check for core mode.
    core_mode = False
    if getattr(nbapp, 'core_mode', False) or app_dir.startswith(HERE):
        core_mode = True
        logger.info('Running JupyterLab in core mode')

    # Check for dev mode.
    dev_mode = False
    if getattr(nbapp, 'dev_mode', False) or app_dir.startswith(DEV_DIR):
        dev_mode = True
        logger.info('Running JupyterLab in dev mode')

    # Check for watch.
    watch_mode = getattr(nbapp, 'watch', False)

    if watch_mode and core_mode:
        logger.warn('Cannot watch in core mode, did you mean --dev-mode?')
        watch_mode = False

    if core_mode and dev_mode:
        logger.warn('Conflicting modes, choosing dev_mode over core_mode')
        core_mode = False

    page_config = web_app.settings.setdefault('page_config_data', dict())
    page_config['buildAvailable'] = not core_mode and not dev_mode
    page_config['buildCheck'] = not core_mode and not dev_mode
    page_config['token'] = nbapp.token
    page_config['devMode'] = dev_mode
    # Export the version info tuple to a JSON array. This get's printed
    # inside double quote marks, so we render it to a JSON string of the
    # JSON data (so that we can call JSON.parse on the frontend on it).
    # We also have to wrap it in `Markup` so that it isn't escaped
    # by Jinja. Otherwise, if the version has string parts these will be
    # escaped and then will have to be unescaped on the frontend.
    page_config['notebookVersion'] = Markup(dumps(dumps(version_info))[1:-1])

    if nbapp.file_to_run and type(nbapp).__name__ == "LabApp":
        relpath = os.path.relpath(nbapp.file_to_run, nbapp.notebook_dir)
        uri = url_escape(ujoin('/lab/tree', *relpath.split(os.sep)))
        nbapp.default_url = uri
        nbapp.file_to_run = ''

    if core_mode:
        app_dir = HERE
        logger.info(CORE_NOTE.strip())
        ensure_core(logger)

    elif dev_mode:
        app_dir = DEV_DIR
        ensure_dev(logger)
        if not watch_mode:
            logger.info(DEV_NOTE)

    config.app_settings_dir = pjoin(app_dir, 'settings')
    config.schemas_dir = pjoin(app_dir, 'schemas')
    config.themes_dir = pjoin(app_dir, 'themes')
    config.workspaces_dir = workspaces_dir
    info = get_app_info(app_dir)
    config.app_version = info['version']
    public_url = info['publicUrl']
    if public_url:
        config.public_url = public_url
    else:
        config.static_dir = pjoin(app_dir, 'static')

    config.user_settings_dir = user_settings_dir

    # The templates end up in the built static directory.
    config.templates_dir = pjoin(app_dir, 'static')

    if watch_mode:
        logger.info('Starting JupyterLab watch mode...')

        # Set the ioloop in case the watch fails.
        nbapp.ioloop = IOLoop.current()
        if dev_mode:
            watch_dev(logger)
        else:
            watch(app_dir, logger)
            page_config['buildAvailable'] = False

        config.cache_files = False

    base_url = web_app.settings['base_url']

    builder = Builder(logger, core_mode, app_dir)
    # build_handler = (build_url, BuildHandler, {'builder': builder})
    handlers = [
        (ujoin(base_url, build_path), BuildHandler, {'builder': builder}),
        (ujoin(base_url, m_build_path), Models_Handler)
    ]

    web_app.add_handlers('.*$', handlers)

    add_handlers(web_app, config)
```

```python
# commands.py
# coding: utf-8
"""JupyterLab command handler"""

# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.
from __future__ import print_function

from distutils.version import LooseVersion
import errno
import glob
import hashlib
import json
import logging
import os
import os.path as osp
import re
import shutil
import site
import sys
import tarfile
from threading import Event

from ipython_genutils.tempdir import TemporaryDirectory
from jupyter_core.paths import jupyter_config_path
from jupyterlab_launcher.process import which, Process, WatchHelper
from notebook.nbextensions import GREEN_ENABLED, GREEN_OK, RED_DISABLED, RED_X

from .semver import Range, gte, lt, lte, gt, make_semver
from .jlpmapp import YARN_PATH, HERE

if sys.version_info.major < 3:
    from urllib2 import Request, urlopen, quote
    from urllib2 import URLError, HTTPError
    from urlparse import urljoin

else:
    from urllib.request import Request, urlopen, urljoin, quote
    from urllib.error import URLError, HTTPError


# The regex for expecting the webpack output.
WEBPACK_EXPECT = re.compile(r'.*/index.out.js')

# The dev mode directory.
DEV_DIR = osp.realpath(os.path.join(HERE, '..', 'dev_mode'))


def pjoin(*args):
    """Join paths to create a real path.
    """
    return osp.realpath(osp.join(*args))


def get_user_settings_dir():
    """Get the configured JupyterLab user settings directory.
    """
    settings_dir = os.environ.get('JUPYTERLAB_SETTINGS_DIR')
    settings_dir = settings_dir or pjoin(
        jupyter_config_path()[0], 'lab', 'user-settings'
    )
    return osp.realpath(settings_dir)


def get_workspaces_dir():
    """Get the configured JupyterLab workspaces directory.
    """
    workspaces_dir = os.environ.get('JUPYTERLAB_WORKSPACES_DIR')
    workspaces_dir = workspaces_dir or pjoin(
        jupyter_config_path()[0], 'lab', 'workspaces'
    )
    return osp.realpath(workspaces_dir)


def get_app_dir():
    """Get the configured JupyterLab app directory.
    """
    # Default to the override environment variable.
    if os.environ.get('JUPYTERLAB_DIR'):
        return osp.realpath(os.environ['JUPYTERLAB_DIR'])

    # Use the default locations for data_files.
    app_dir = pjoin(sys.prefix, 'share', 'jupyter', 'lab')

    # Check for a user level install.
    # Ensure that USER_BASE is defined
    if hasattr(site, 'getuserbase'):
        site.getuserbase()
    userbase = getattr(site, 'USER_BASE', None)
    if HERE.startswith(userbase) and not app_dir.startswith(userbase):
        app_dir = pjoin(userbase, 'share', 'jupyter', 'lab')

    # Check for a system install in '/usr/local/share'.
    elif (sys.prefix.startswith('/usr') and not
          osp.exists(app_dir) and
          osp.exists('/usr/local/share/jupyter/lab')):
        app_dir = '/usr/local/share/jupyter/lab'

    return osp.realpath(app_dir)


def ensure_dev(logger=None):
    """Ensure that the dev assets are available.
    """
    parent = pjoin(HERE, '..')

    if not osp.exists(pjoin(parent, 'node_modules')):
        yarn_proc = Process(['node', YARN_PATH], cwd=parent, logger=logger)
        yarn_proc.wait()

    theme_packages = ['theme-light-extension', 'theme-dark-extension']
    for theme in theme_packages:
        base_path = pjoin(parent, 'packages', theme)
        if not osp.exists(pjoin(base_path, 'static')):
            yarn_proc = Process(['node', YARN_PATH, 'build:webpack'], cwd=base_path,
                                logger=logger)
            yarn_proc.wait()

    if not osp.exists(pjoin(parent, 'dev_mode', 'static')):
        yarn_proc = Process(['node', YARN_PATH, 'build'], cwd=parent,
                            logger=logger)
        yarn_proc.wait()


def ensure_core(logger=None):
    """Ensure that the core assets are available.
    """
    staging = pjoin(HERE, 'staging')

    # Bail if the static directory already exists.
    if osp.exists(pjoin(HERE, 'static')):
        return

    if not osp.exists(pjoin(staging, 'node_modules')):
        yarn_proc = Process(['node', YARN_PATH], cwd=staging, logger=logger)
        yarn_proc.wait()

    if not osp.exists(pjoin(HERE, 'static')):
        yarn_proc = Process(['node', YARN_PATH, 'build'], cwd=staging,
                            logger=logger)
        yarn_proc.wait()


def watch_packages(logger=None):
    """Run watch mode for the source packages.

    Parameters
    ----------
    logger: :class:`~logger.Logger`, optional
        The logger instance.

    Returns
    -------
    A list of `WatchHelper` objects.
    """
    parent = pjoin(HERE, '..')

    if not osp.exists(pjoin(parent, 'node_modules')):
        yarn_proc = Process(['node', YARN_PATH], cwd=parent, logger=logger)
        yarn_proc.wait()

    logger = logger or logging.getLogger('jupyterlab')
    ts_dir = osp.realpath(osp.join(HERE, '..', 'packages', 'metapackage'))

    # Run typescript watch and wait for the string indicating it is done.
    ts_regex = r'.* Found 0 errors\. Watching for file changes\.'
    ts_proc = WatchHelper(['node', YARN_PATH, 'run', 'watch'],
        cwd=ts_dir, logger=logger, startup_regex=ts_regex)

    # Run the metapackage file watcher.
    tsf_regex = 'Watching the metapackage files...'
    tsf_proc = WatchHelper(['node', YARN_PATH, 'run', 'watch:files'],
        cwd=ts_dir, logger=logger, startup_regex=tsf_regex)

    return [ts_proc, tsf_proc]


def watch_dev(logger=None):
    """Run watch mode in a given directory.

    Parameters
    ----------
    logger: :class:`~logger.Logger`, optional
        The logger instance.

    Returns
    -------
    A list of `WatchHelper` objects.
    """
    logger = logger or logging.getLogger('jupyterlab')

    package_procs = watch_packages(logger)

    # Run webpack watch and wait for compilation.
    wp_proc = WatchHelper(['node', YARN_PATH, 'run', 'watch'],
        cwd=DEV_DIR, logger=logger,
        startup_regex=WEBPACK_EXPECT)

    return package_procs + [wp_proc]


def watch(app_dir=None, logger=None):
    """Watch the application.

    Parameters
    ----------
    app_dir: string, optional
        The application directory.
    logger: :class:`~logger.Logger`, optional
        The logger instance.

    Returns
    -------
    A list of processes to run asynchronously.
    """
    _node_check()
    handler = _AppHandler(app_dir, logger)
    return handler.watch()


def install_extension(extension, app_dir=None, logger=None):
    """Install an extension package into JupyterLab.

    The extension is first validated.

    Returns `True` if a rebuild is recommended, `False` otherwise.
    """
    _node_check()
    handler = _AppHandler(app_dir, logger)
    return handler.install_extension(extension)


def uninstall_extension(name, app_dir=None, logger=None):
    """Uninstall an extension by name or path.

    Returns `True` if a rebuild is recommended, `False` otherwise.
    """
    _node_check()
    handler = _AppHandler(app_dir, logger)
    return handler.uninstall_extension(name)


def update_extension(name=None, all_=False, app_dir=None, logger=None):
    """Update an extension by name, or all extensions.

    Either `name` must be given as a string, or `all_` must be `True`.
    If `all_` is `True`, the value of `name` is ignored.

    Returns `True` if a rebuild is recommended, `False` otherwise.
    """
    _node_check()
    handler = _AppHandler(app_dir, logger)
    if all_ is True:
        return handler.update_all_extensions()
    return handler.update_extension(name)


def clean(app_dir=None):
    """Clean the JupyterLab application directory."""
    app_dir = app_dir or get_app_dir()
    if app_dir == pjoin(HERE, 'dev'):
        raise ValueError('Cannot clean the dev app')
    if app_dir == pjoin(HERE, 'core'):
        raise ValueError('Cannot clean the core app')
    for name in ['staging']:
        target = pjoin(app_dir, name)
        if osp.exists(target):
            shutil.rmtree(target)


def build(app_dir=None, name=None, version=None, public_url=None,
        logger=None, command='build:prod', kill_event=None,
        clean_staging=False):
    """Build the JupyterLab application.
    """
    _node_check()
    handler = _AppHandler(app_dir, logger, kill_event=kill_event)
    return handler.build(name=name, version=version, public_url=public_url,
                  command=command, clean_staging=clean_staging)


def get_app_info(app_dir=None, logger=None):
    """Get a dictionary of information about the app.
    """
    handler = _AppHandler(app_dir, logger)
    return handler.info


def enable_extension(extension, app_dir=None, logger=None):
    """Enable a JupyterLab extension.

    Returns `True` if a rebuild is recommended, `False` otherwise.
    """
    handler = _AppHandler(app_dir, logger)
    return handler.toggle_extension(extension, False)


def disable_extension(extension, app_dir=None, logger=None):
    """Disable a JupyterLab package.

    Returns `True` if a rebuild is recommended, `False` otherwise.
    """
    handler = _AppHandler(app_dir, logger)
    return handler.toggle_extension(extension, True)


def check_extension(extension, app_dir=None, installed=False, logger=None):
    """Check if a JupyterLab extension is enabled or disabled.
    """
    handler = _AppHandler(app_dir, logger)
    return handler.check_extension(extension, installed)


def build_check(app_dir=None, logger=None):
    """Determine whether JupyterLab should be built.

    Returns a list of messages.
    """
    _node_check()
    handler = _AppHandler(app_dir, logger)
    return handler.build_check()


def list_extensions(app_dir=None, logger=None):
    """List the extensions.
    """
    handler = _AppHandler(app_dir, logger)
    return handler.list_extensions()


def link_package(path, app_dir=None, logger=None):
    """Link a package against the JupyterLab build.

    Returns `True` if a rebuild is recommended, `False` otherwise.
    """
    handler = _AppHandler(app_dir, logger)
    return handler.link_package(path)


def unlink_package(package, app_dir=None, logger=None):
    """Unlink a package from JupyterLab by path or name.

    Returns `True` if a rebuild is recommended, `False` otherwise.
    """
    handler = _AppHandler(app_dir, logger)
    return handler.unlink_package(package)


def get_app_version(app_dir=None):
    """Get the application version."""
    app_dir = app_dir or get_app_dir()
    handler = _AppHandler(app_dir)
    return handler.info['version']


# ----------------------------------------------------------------------
# Implementation details
# ----------------------------------------------------------------------


class _AppHandler(object):

    def __init__(self, app_dir, logger=None, kill_event=None):
        """Create a new _AppHandler object
        """
        self.app_dir = app_dir or get_app_dir()
        self.sys_dir = get_app_dir()
        self.logger = logger or logging.getLogger('jupyterlab')
        self.info = self._get_app_info()
        self.kill_event = kill_event or Event()
        # TODO: Make this configurable
        self.registry = 'https://registry.npmjs.org'

    def install_extension(self, extension, existing=None):
        """Install an extension package into JupyterLab.

        The extension is first validated.

        Returns `True` if a rebuild is recommended, `False` otherwise.
        """
        extension = _normalize_path(extension)
        extensions = self.info['extensions']

        # Check for a core extensions.
        if extension in self.info['core_extensions']:
            config = self._read_build_config()
            uninstalled = config.get('uninstalled_core_extensions', [])
            if extension in uninstalled:
                self.logger.info('Installing core extension %s' % extension)
                uninstalled.remove(extension)
                config['uninstalled_core_extensions'] = uninstalled
                self._write_build_config(config)
                return True
            return False

        # Create the app dirs if needed.
        self._ensure_app_dirs()

        # Install the package using a temporary directory.
        with TemporaryDirectory() as tempdir:
            info = self._install_extension(extension, tempdir)

        name = info['name']

        # Local directories get name mangled and stored in metadata.
        if info['is_dir']:
            config = self._read_build_config()
            local = config.setdefault('local_extensions', dict())
            local[name] = info['source']
            self._write_build_config(config)

        # Remove an existing extension with the same name and different path
        if name in extensions:
            other = extensions[name]
            if other['path'] != info['path'] and other['location'] == 'app':
                os.remove(other['path'])

        return True

    def build(self, name=None, version=None, public_url=None,
            command='build:prod', clean_staging=False):
        """Build the application.
        """
        # Set up the build directory.
        app_dir = self.app_dir

        self._populate_staging(
            name=name, version=version, public_url=public_url,
            clean=clean_staging
        )

        staging = pjoin(app_dir, 'staging')

        # Make sure packages are installed.
        self._run(['node', YARN_PATH, 'install'], cwd=staging)

        # Build the app.
        self._run(['node', YARN_PATH, 'run', command], cwd=staging)

    def watch(self):
        """Start the application watcher and then run the watch in
        the background.
        """
        staging = pjoin(self.app_dir, 'staging')

        self._populate_staging()

        # Make sure packages are installed.
        self._run(['node', YARN_PATH, 'install'], cwd=staging)

        proc = WatchHelper(['node', YARN_PATH, 'run', 'watch'],
            cwd=pjoin(self.app_dir, 'staging'),
            startup_regex=WEBPACK_EXPECT,
            logger=self.logger)
        return [proc]

    def list_extensions(self):
        """Print an output of the extensions.
        """
        logger = self.logger
        info = self.info

        logger.info('JupyterLab v%s' % info['version'])

        if info['extensions']:
            info['compat_errors'] = self._get_extension_compat()
            logger.info('Known labextensions:')
            self._list_extensions(info, 'app')
            self._list_extensions(info, 'sys')
        else:
            logger.info('No installed extensions')

        local = info['local_extensions']
        if local:
            logger.info('\n   local extensions:')
            for name in sorted(local):
                logger.info('        %s: %s' % (name, local[name]))

        linked_packages = info['linked_packages']
        if linked_packages:
            logger.info('\n   linked packages:')
            for key in sorted(linked_packages):
                source = linked_packages[key]['source']
                logger.info('        %s: %s' % (key, source))

        uninstalled_core = info['uninstalled_core']
        if uninstalled_core:
            logger.info('\nUninstalled core extensions:')
            [logger.info('    %s' % item) for item in sorted(uninstalled_core)]

        disabled_core = info['disabled_core']
        if disabled_core:
            logger.info('\nDisabled core extensions:')
            [logger.info('    %s' % item) for item in sorted(disabled_core)]

        messages = self.build_check(fast=True)
        if messages:
            logger.info('\nBuild recommended, please run `jupyter lab build`:')
            [logger.info('    %s' % item) for item in messages]

    def build_check(self, fast=False):
        """Determine whether JupyterLab should be built.

        Returns a list of messages.
        """
        app_dir = self.app_dir
        local = self.info['local_extensions']
        linked = self.info['linked_packages']
        messages = []

        # Check for no application.
        pkg_path = pjoin(app_dir, 'static', 'package.json')
        if not osp.exists(pkg_path):
            return ['No built application']

        static_data = self.info['static_data']
        old_jlab = static_data['jupyterlab']
        old_deps = static_data.get('dependencies', dict())

        # Look for mismatched version.
        static_version = old_jlab.get('version', '')
        core_version = old_jlab['version']
        if LooseVersion(static_version) != LooseVersion(core_version):
            msg = 'Version mismatch: %s (built), %s (current)'
            return [msg % (static_version, core_version)]

        # Look for mismatched extensions.
        new_package = self._get_package_template(silent=fast)
        new_jlab = new_package['jupyterlab']
        new_deps = new_package.get('dependencies', dict())

        for ext_type in ['extensions', 'mimeExtensions']:
            # Extensions that were added.
            for ext in new_jlab[ext_type]:
                if ext not in old_jlab[ext_type]:
                    messages.append('%s needs to be included in build' % ext)

            # Extensions that were removed.
            for ext in old_jlab[ext_type]:
                if ext not in new_jlab[ext_type]:
                    messages.append('%s needs to be removed from build' % ext)

        # Look for mismatched dependencies
        for (pkg, dep) in new_deps.items():
            if pkg not in old_deps:
                continue
            # Skip local and linked since we pick them up separately.
            if pkg in local or pkg in linked:
                continue
            if old_deps[pkg] != dep:
                msg = '%s changed from %s to %s'
                messages.append(msg % (pkg, old_deps[pkg], new_deps[pkg]))

        # Look for updated local extensions.
        for (name, source) in local.items():
            if fast:
                continue
            dname = pjoin(app_dir, 'extensions')
            if self._check_local(name, source, dname):
                messages.append('%s content changed' % name)

        # Look for updated linked packages.
        for (name, item) in linked.items():
            if fast:
                continue
            dname = pjoin(app_dir, 'staging', 'linked_packages')
            if self._check_local(name, item['source'], dname):
                messages.append('%s content changed' % name)

        return messages

    def uninstall_extension(self, name):
        """Uninstall an extension by name.

        Returns `True` if a rebuild is recommended, `False` otherwise.
        """
        # Allow for uninstalled core extensions.
        data = self.info['core_data']
        if name in self.info['core_extensions']:
            config = self._read_build_config()
            uninstalled = config.get('uninstalled_core_extensions', [])
            if name not in uninstalled:
                self.logger.info('Uninstalling core extension %s' % name)
                uninstalled.append(name)
                config['uninstalled_core_extensions'] = uninstalled
                self._write_build_config(config)
                return True
            return False

        local = self.info['local_extensions']

        for (extname, data) in self.info['extensions'].items():
            path = data['path']
            if extname == name:
                msg = 'Uninstalling %s from %s' % (name, osp.dirname(path))
                self.logger.info(msg)
                os.remove(path)
                # Handle local extensions.
                if extname in local:
                    config = self._read_build_config()
                    data = config.setdefault('local_extensions', dict())
                    del data[extname]
                    self._write_build_config(config)
                return True

        self.logger.warn('No labextension named "%s" installed' % name)
        return False

    def update_all_extensions(self):
        """Update all non-local extensions.

        Returns `True` if a rebuild is recommended, `False` otherwise.
        """
        should_rebuild = False
        for (extname, _) in self.info['extensions'].items():
            if extname in self.info['local_extensions']:
                continue
            updated = self._update_extension(extname)
            # Rebuild if at least one update happens:
            should_rebuild = should_rebuild or updated
        return should_rebuild

    def update_extension(self, name):
        """Update an extension by name.

        Returns `True` if a rebuild is recommended, `False` otherwise.
        """
        if name not in self.info['extensions']:
            self.logger.warn('No labextension named "%s" installed' % name)
            return False
        return self._update_extension(name)

    def _update_extension(self, name):
        """Update an extension by name.

        Returns `True` if a rebuild is recommended, `False` otherwise.
        """
        try:
            latest = self._latest_compatible_package_version(name)
        except URLError:
            return False
        if latest is None:
            return False
        if latest == self.info['extensions'][name]['version']:
            self.logger.info('Extension %r already up to date' % name)
            return False
        self.logger.info('Updating %s to version %s' % (name, latest))
        return self.install_extension('%s@%s' % (name, latest))


    def link_package(self, path):
        """Link a package at the given path.

        Returns `True` if a rebuild is recommended, `False` otherwise.
        """
        path = _normalize_path(path)
        if not osp.exists(path) or not osp.isdir(path):
            msg = 'Can install "%s" only link local directories'
            raise ValueError(msg % path)

        with TemporaryDirectory() as tempdir:
            info = self._extract_package(path, tempdir)

        messages = _validate_extension(info['data'])
        if not messages:
            return self.install_extension(path)

        # Warn that it is a linked package.
        self.logger.warn('Installing %s as a linked package:', path)
        [self.logger.warn(m) for m in messages]

        # Add to metadata.
        config = self._read_build_config()
        linked = config.setdefault('linked_packages', dict())
        linked[info['name']] = info['source']
        self._write_build_config(config)

        return True

    def unlink_package(self, path):
        """Unlink a package by name or at the given path.

        A ValueError is raised if the path is not an unlinkable package.

        Returns `True` if a rebuild is recommended, `False` otherwise.
        """
        path = _normalize_path(path)
        config = self._read_build_config()
        linked = config.setdefault('linked_packages', dict())

        found = None
        for (name, source) in linked.items():
            if name == path or source == path:
                found = name

        if found:
            del linked[found]
        else:
            local = config.setdefault('local_extensions', dict())
            for (name, source) in local.items():
                if name == path or source == path:
                    found = name
            if found:
                del local[found]
                path = self.info['extensions'][found]['path']
                os.remove(path)

        if not found:
            raise ValueError('No linked package for %s' % path)

        self._write_build_config(config)
        return True

    def toggle_extension(self, extension, value):
        """Enable or disable a lab extension.

        Returns `True` if a rebuild is recommended, `False` otherwise.
        """
        config = self._read_page_config()
        disabled = config.setdefault('disabledExtensions', [])
        did_something = False
        if value and extension not in disabled:
            disabled.append(extension)
            did_something = True
        elif not value and extension in disabled:
            disabled.remove(extension)
            did_something = True
        if did_something:
            self._write_page_config(config)
        return did_something

    def check_extension(self, extension, check_installed_only=False):
        """Check if a lab extension is enabled or disabled
        """
        info = self.info

        if extension in info["core_extensions"]:
            return self._check_core_extension(
                extension, info, check_installed_only)

        if extension in info["linked_packages"]:
            self.logger.info('%s:%s' % (extension, GREEN_ENABLED))
            return True

        return self._check_common_extension(
            extension, info, check_installed_only)

    def _check_core_extension(self, extension, info, check_installed_only):
        """Check if a core extension is enabled or disabled
        """
        if extension in info['uninstalled_core']:
            self.logger.info('%s:%s' % (extension, RED_X))
            return False
        if check_installed_only:
            self.logger.info('%s: %s' % (extension, GREEN_OK))
            return True
        if extension in info['disabled_core']:
            self.logger.info('%s: %s' % (extension, RED_DISABLED))
            return False
        self.logger.info('%s:%s' % (extension, GREEN_ENABLED))
        return True

    def _check_common_extension(self, extension, info, check_installed_only):
        """Check if a common (non-core) extension is enabled or disabled
        """
        if extension not in info['extensions']:
            self.logger.info('%s:%s' % (extension, RED_X))
            return False

        errors = self._get_extension_compat()[extension]
        if errors:
            self.logger.info('%s:%s (compatibility errors)' % (extension, RED_X))
            return False

        if check_installed_only:
            self.logger.info('%s: %s' % (extension, GREEN_OK))
            return True

        if _is_disabled(extension, info['disabled']):
            self.logger.info('%s: %s' % (extension, RED_DISABLED))
            return False

        self.logger.info('%s:%s' % (extension, GREEN_ENABLED))
        return True

    def _get_app_info(self):
        """Get information about the app.
        """

        info = dict()
        info['core_data'] = core_data = _get_core_data()
        info['extensions'] = extensions = self._get_extensions(core_data)
        page_config = self._read_page_config()
        info['disabled'] = page_config.get('disabledExtensions', [])
        info['local_extensions'] = self._get_local_extensions()
        info['linked_packages'] = self._get_linked_packages()
        info['app_extensions'] = app = []
        info['sys_extensions'] = sys = []
        for (name, data) in extensions.items():
            data['is_local'] = name in info['local_extensions']
            if data['location'] == 'app':
                app.append(name)
            else:
                sys.append(name)

        info['uninstalled_core'] = self._get_uninstalled_core_extensions()

        info['static_data'] = _get_static_data(self.app_dir)
        app_data = info['static_data'] or core_data
        info['version'] = app_data['jupyterlab']['version']
        info['publicUrl'] = app_data['jupyterlab'].get('publicUrl', '')

        info['sys_dir'] = self.sys_dir
        info['app_dir'] = self.app_dir

        info['core_extensions'] = core_extensions = _get_core_extensions()

        disabled_core = []
        for key in core_extensions:
            if key in info['disabled']:
                disabled_core.append(key)

        info['disabled_core'] = disabled_core
        return info

    def _populate_staging(self, name=None, version=None, public_url=None,
            clean=False):
        """Set up the assets in the staging directory.
        """
        app_dir = self.app_dir
        staging = pjoin(app_dir, 'staging')
        if clean and osp.exists(staging):
            self.logger.info("Cleaning %s", staging)
            shutil.rmtree(staging)

        self._ensure_app_dirs()
        if not version:
            version = self.info['core_data']['jupyterlab']['version']

        # Look for mismatched version.
        pkg_path = pjoin(staging, 'package.json')

        if osp.exists(pkg_path):
            with open(pkg_path) as fid:
                data = json.load(fid)
            if data['jupyterlab'].get('version', '') != version:
                shutil.rmtree(staging)
                os.makedirs(staging)

        for fname in ['index.js', 'webpack.config.js',
                      'webpack.prod.config.js',
                      '.yarnrc', 'yarn.js']:
            target = pjoin(staging, fname)
            shutil.copy(pjoin(HERE, 'staging', fname), target)

        # Remove an existing yarn.lock file
        # Because otherwise we can end up with unwanted duplicates
        # cf https://github.com/yarnpkg/yarn/issues/3967
        if osp.exists(pjoin(staging, 'yarn.lock')):
            os.remove(pjoin(staging, 'yarn.lock'))

        # Ensure a clean templates directory
        templates = pjoin(staging, 'templates')
        if osp.exists(templates):
            shutil.rmtree(templates)
        shutil.copytree(pjoin(HERE, 'staging', 'templates'), templates)

        # Ensure a clean linked packages directory.
        linked_dir = pjoin(staging, 'linked_packages')
        if osp.exists(linked_dir):
            shutil.rmtree(linked_dir)
        os.makedirs(linked_dir)

        # Template the package.json file.
        # Update the local extensions.
        extensions = self.info['extensions']
        removed = False
        for (key, source) in self.info['local_extensions'].items():
            # Handle a local extension that was removed.
            if key not in extensions:
                config = self._read_build_config()
                data = config.setdefault('local_extensions', dict())
                del data[key]
                self._write_build_config(config)
                removed = True
                continue
            dname = pjoin(app_dir, 'extensions')
            self._update_local(key, source, dname, extensions[key],
                'local_extensions')

        # Update the list of local extensions if any were removed.
        if removed:
            self.info['local_extensions'] = self._get_local_extensions()

        # Update the linked packages.
        linked = self.info['linked_packages']
        for (key, item) in linked.items():
            dname = pjoin(staging, 'linked_packages')
            self._update_local(key, item['source'], dname, item,
                'linked_packages')

        # Then get the package template.
        data = self._get_package_template()

        if version:
            data['jupyterlab']['version'] = version

        if name:
            data['jupyterlab']['name'] = name

        if public_url:
            data['jupyterlab']['publicUrl'] = public_url

        pkg_path = pjoin(staging, 'package.json')
        with open(pkg_path, 'w') as fid:
            json.dump(data, fid, indent=4)

    def _get_package_template(self, silent=False):
        """Get the template the for staging package.json file.
        """
        logger = self.logger
        data = self.info['core_data']
        local = self.info['local_extensions']
        linked = self.info['linked_packages']
        extensions = self.info['extensions']
        jlab = data['jupyterlab']

        def format_path(path):
            path = osp.relpath(path, pjoin(self.app_dir, 'staging'))
            path = 'file:' + path.replace(os.sep, '/')
            if os.name == 'nt':
                path = path.lower()
            return path

        jlab['linkedPackages'] = dict()

        # Handle local extensions.
        for (key, source) in local.items():
            jlab['linkedPackages'][key] = source

        # Handle linked packages.
        for (key, item) in linked.items():
            path = pjoin(self.app_dir, 'staging', 'linked_packages')
            path = pjoin(path, item['filename'])
            data['dependencies'][key] = format_path(path)
            jlab['linkedPackages'][key] = item['source']

        # Handle extensions
        compat_errors = self._get_extension_compat()
        for (key, value) in extensions.items():
            # Reject incompatible extensions with a message.
            errors = compat_errors[key]
            if errors:
                msg = _format_compatibility_errors(
                    key, value['version'], errors
                )
                if not silent:
                    logger.warn(msg + '\n')
                continue

            data['dependencies'][key] = format_path(value['path'])

            jlab_data = value['jupyterlab']
            for item in ['extension', 'mimeExtension']:
                ext = jlab_data.get(item, False)
                if not ext:
                    continue
                if ext is True:
                    ext = ''
                jlab[item + 's'][key] = ext

        # Handle uninstalled core extensions.
        for item in self.info['uninstalled_core']:
            if item in jlab['extensions']:
                data['jupyterlab']['extensions'].pop(item)
            else:
                data['jupyterlab']['mimeExtensions'].pop(item)
            # Remove from dependencies as well.
            data['dependencies'].pop(item)

        return data

    def _check_local(self, name, source, dname):
        """Check if a local package has changed.

        `dname` is the directory name of existing package tar archives.
        """
        # Extract the package in a temporary directory.
        with TemporaryDirectory() as tempdir:
            info = self._extract_package(source, tempdir)
            # Test if the file content has changed.
            # This relies on `_extract_package` adding the hashsum
            # to the filename, allowing a simple exist check to
            # compare the hash to the "cache" in dname.
            target = pjoin(dname, info['filename'])
            return not osp.exists(target)

    def _update_local(self, name, source, dname, data, dtype):
        """Update a local dependency.  Return `True` if changed.
        """
        # Extract the package in a temporary directory.
        existing = data['filename']
        with TemporaryDirectory() as tempdir:
            info = self._extract_package(source, tempdir)

            # Bail if the file content has not changed.
            if info['filename'] == existing:
                return existing

            shutil.move(info['path'], pjoin(dname, info['filename']))

        # Remove the existing tarball and return the new file name.
        if existing:
            os.remove(pjoin(dname, existing))

        data['filename'] = info['filename']
        data['path'] = pjoin(data['tar_dir'], data['filename'])
        return info['filename']

    def _get_extensions(self, core_data):
        """Get the extensions for the application.
        """
        app_dir = self.app_dir
        extensions = dict()

        # Get system level packages.
        sys_path = pjoin(self.sys_dir, 'extensions')
        app_path = pjoin(self.app_dir, 'extensions')

        extensions = self._get_extensions_in_dir(self.sys_dir, core_data)

        # Look in app_dir if different.
        app_path = pjoin(app_dir, 'extensions')
        if app_path == sys_path or not osp.exists(app_path):
            return extensions

        extensions.update(self._get_extensions_in_dir(app_dir, core_data))

        return extensions

    def _get_extensions_in_dir(self, dname, core_data):
        """Get the extensions in a given directory.
        """
        extensions = dict()
        location = 'app' if dname == self.app_dir else 'sys'
        for target in glob.glob(pjoin(dname, 'extensions', '*.tgz')):
            data = _read_package(target)
            deps = data.get('dependencies', dict())
            name = data['name']
            jlab = data.get('jupyterlab', dict())
            path = osp.realpath(target)
            extensions[name] = dict(path=path,
                                    filename=osp.basename(path),
                                    version=data['version'],
                                    jupyterlab=jlab,
                                    dependencies=deps,
                                    tar_dir=osp.dirname(path),
                                    location=location)
        return extensions

    def _get_extension_compat(self):
        """Get the extension compatibility info.
        """
        compat = dict()
        core_data = self.info['core_data']
        for (name, data) in self.info['extensions'].items():
            deps = data['dependencies']
            compat[name] = _validate_compatibility(name, deps, core_data)
        return compat

    def _get_local_extensions(self):
        """Get the locally installed extensions.
        """
        return self._get_local_data('local_extensions')

    def _get_linked_packages(self):
        """Get the linked packages.
        """
        info = self._get_local_data('linked_packages')
        dname = pjoin(self.app_dir, 'staging', 'linked_packages')
        for (name, source) in info.items():
            info[name] = dict(source=source, filename='', tar_dir=dname)

        if not osp.exists(dname):
            return info

        for path in glob.glob(pjoin(dname, '*.tgz')):
            path = osp.realpath(path)
            data = _read_package(path)
            name = data['name']
            if name not in info:
                self.logger.warn('Removing orphaned linked package %s' % name)
                os.remove(path)
                continue
            item = info[name]
            item['filename'] = osp.basename(path)
            item['path'] = path
            item['version'] = data['version']
            item['data'] = data
        return info

    def _get_uninstalled_core_extensions(self):
        """Get the uninstalled core extensions.
        """
        config = self._read_build_config()
        return config.get('uninstalled_core_extensions', [])

    def _ensure_app_dirs(self):
        """Ensure that the application directories exist"""
        dirs = ['extensions', 'settings', 'staging', 'schemas', 'themes']
        for dname in dirs:
            path = pjoin(self.app_dir, dname)
            if not osp.exists(path):
                try:
                    os.makedirs(path)
                except OSError as e:
                    if e.errno != errno.EEXIST:
                        raise

    def _list_extensions(self, info, ext_type):
        """List the extensions of a given type.
        """
        logger = self.logger
        names = info['%s_extensions' % ext_type]
        if not names:
            return

        dname = info['%s_dir' % ext_type]

        logger.info('   %s dir: %s' % (ext_type, dname))
        for name in sorted(names):
            logger.info(name)
            data = info['extensions'][name]
            version = data['version']
            errors = info['compat_errors'][name]
            extra = ''
            if _is_disabled(name, info['disabled']):
                extra += ' %s' % RED_DISABLED
            else:
                extra += ' %s' % GREEN_ENABLED
            if errors:
                extra += ' %s' % RED_X
            else:
                extra += ' %s' % GREEN_OK
            if data['is_local']:
                extra += '*'
            logger.info('        %s v%s%s' % (name, version, extra))
            if errors:
                msg = _format_compatibility_errors(
                    name, version, errors
                )
                logger.warn(msg + '\n')

    def _read_build_config(self):
        """Get the build config data for the app dir.
        """
        target = pjoin(self.app_dir, 'settings', 'build_config.json')
        if not osp.exists(target):
            return {}
        else:
            with open(target) as fid:
                return json.load(fid)

    def _write_build_config(self, config):
        """Write the build config to the app dir.
        """
        self._ensure_app_dirs()
        target = pjoin(self.app_dir, 'settings', 'build_config.json')
        with open(target, 'w') as fid:
            json.dump(config, fid, indent=4)

    def _read_page_config(self):
        """Get the page config data for the app dir.
        """
        target = pjoin(self.app_dir, 'settings', 'page_config.json')
        if not osp.exists(target):
            return {}
        else:
            with open(target) as fid:
                return json.load(fid)

    def _write_page_config(self, config):
        """Write the build config to the app dir.
        """
        self._ensure_app_dirs()
        target = pjoin(self.app_dir, 'settings', 'page_config.json')
        with open(target, 'w') as fid:
            json.dump(config, fid, indent=4)

    def _get_local_data(self, source):
        """Get the local data for extensions or linked packages.
        """
        config = self._read_build_config()

        data = config.setdefault(source, dict())
        dead = []
        for (name, source) in data.items():
            if not osp.exists(source):
                dead.append(name)

        for name in dead:
            link_type = source.replace('_', ' ')
            msg = '**Note: Removing dead %s "%s"' % (link_type, name)
            self.logger.warn(msg)
            del data[name]

        if dead:
            self._write_build_config(config)

        return data

    def _install_extension(self, extension, tempdir):
        """Install an extension with validation and return the name and path.
        """
        info = self._extract_package(extension, tempdir)
        data = info['data']

        # Verify that the package is an extension.
        messages = _validate_extension(data)
        if messages:
            msg = '"%s" is not a valid extension:\n%s'
            raise ValueError(msg % (extension, '\n'.join(messages)))

        # Verify package compatibility.
        core_data = _get_core_data()
        deps = data.get('dependencies', dict())
        errors = _validate_compatibility(extension, deps, core_data)
        if errors:
            msg = _format_compatibility_errors(
                data['name'], data['version'], errors
            )
            # Check for compatible version unless:
            # - A specific version was requested (@ in name,
            #   but after first char to allow for scope marker).
            # - Package is locally installed.
            if '@' not in extension[1:] and not info['is_dir']:
                name = info['name']
                try:
                    version = self._latest_compatible_package_version(name)
                except URLError:
                    # We cannot add any additional information to error message
                    raise ValueError(msg)

                if version and name:
                    self.logger.warning('Incompatible extension:\n%s', msg)
                    self.logger.warning('Found compatible version: %s', version)
                    with TemporaryDirectory() as tempdir2:
                        return self._install_extension(
                            '%s@%s' % (name, version), tempdir2)

                # Extend message to better guide the user what to do:
                conflicts = '\n'.join(msg.splitlines()[2:])
                msg = ''.join((
                    self._format_no_compatible_package_version(name),
                    "\n\n",
                    conflicts))

            raise ValueError(msg)

        # Move the file to the app directory.
        target = pjoin(self.app_dir, 'extensions', info['filename'])
        if osp.exists(target):
            os.remove(target)

        shutil.move(info['path'], target)

        info['path'] = target
        return info

    def _extract_package(self, source, tempdir, quiet=False):
        """Call `npm pack` for an extension.

        The pack command will download the package tar if `source` is
        a package name, or run `npm pack` locally if `source` is a
        directory.
        """
        is_dir = osp.exists(source) and osp.isdir(source)
        if is_dir and not osp.exists(pjoin(source, 'node_modules')):
            self._run(['node', YARN_PATH, 'install'], cwd=source, quiet=quiet)

        info = dict(source=source, is_dir=is_dir)

        ret = self._run([which('npm'), 'pack', source], cwd=tempdir, quiet=quiet)
        if ret != 0:
            msg = '"%s" is not a valid npm package'
            raise ValueError(msg % source)

        path = glob.glob(pjoin(tempdir, '*.tgz'))[0]
        info['data'] = _read_package(path)
        if is_dir:
            info['sha'] = sha = _tarsum(path)
            target = path.replace('.tgz', '-%s.tgz' % sha)
            shutil.move(path, target)
            info['path'] = target
        else:
            info['path'] = path

        info['filename'] = osp.basename(info['path'])
        info['name'] = info['data']['name']
        info['version'] = info['data']['version']

        return info


    def _latest_compatible_package_version(self, name):
        """Get the latest compatible version of a package"""
        core_data = self.info['core_data']
        metadata = _fetch_package_metadata(self.registry, name, self.logger)
        versions = metadata.get('versions', [])

        # Sort pre-release first, as we will reverse the sort:
        def sort_key(key_value):
            return _semver_key(key_value[0], prerelease_first=True)

        for version, data in sorted(versions.items(),
                                    key=sort_key,
                                    reverse=True):
            deps = data.get('dependencies', {})
            errors = _validate_compatibility(name, deps, core_data)
            if not errors:
                # Found a compatible version
                # Verify that the version is a valid extension.
                with TemporaryDirectory() as tempdir:
                    info = self._extract_package(
                        '%s@%s' % (name, version), tempdir, quiet=True)
                if _validate_extension(info['data']):
                    # Invalid, do not consider other versions
                    return
                # Valid
                return version

    def _format_no_compatible_package_version(self, name):
        """Get the latest compatible version of a package"""
        core_data = self.info['core_data']
        metadata = _fetch_package_metadata(self.registry, name, self.logger)
        versions = metadata.get('versions', [])

        # Sort pre-release first, as we will reverse the sort:
        def sort_key(key_value):
            return _semver_key(key_value[0], prerelease_first=True)

        store = tuple(sorted(versions.items(), key=sort_key, reverse=True))
        latest_deps = store[0][1].get('dependencies', {})
        core_deps = core_data['dependencies']
        singletons = core_data['jupyterlab']['singletonPackages']

        # Whether lab version is too new:
        lab_newer_than_latest = False
        # Whether the latest version of the extension depend on a "future" version
        # of a singleton package (from the perspective of current lab version):
        latest_newer_than_lab = False

        for (key, value) in latest_deps.items():
            if key in singletons:
                c = _compare_ranges(core_deps[key], value)
                lab_newer_than_latest = lab_newer_than_latest or c < 0
                latest_newer_than_lab = latest_newer_than_lab or c > 0

        if lab_newer_than_latest:
            # All singleton deps in current version of lab are newer than those
            # in the latest version of the extension
            return ("This extension does not yet support the current version of "
                    "JupyterLab.\n")


        parts = ["No version of {extension} could be found that is compatible with "
                 "the current version of JupyterLab."]
        if latest_newer_than_lab:
            parts.extend(("However, it seems to support a new version of JupyterLab.",
                          "Consider upgrading JupyterLab."))

        return " ".join(parts).format(extension=name)

    def _run(self, cmd, **kwargs):
        """Run the command using our logger and abort callback.

        Returns the exit code.
        """
        if self.kill_event.is_set():
            raise ValueError('Command was killed')

        kwargs['logger'] = self.logger
        kwargs['kill_event'] = self.kill_event
        proc = Process(cmd, **kwargs)
        return proc.wait()


def _node_check():
    """Check for the existence of nodejs with the correct version.
    """
    try:
        proc = Process(['node', 'node-version-check.js'], cwd=HERE, quiet=True)
        proc.wait()
    except Exception:
        msg = 'Please install nodejs 5+ and npm before continuing. nodejs may be installed using conda or directly from the nodejs website.'
        raise ValueError(msg)


def _normalize_path(extension):
    """Normalize a given extension if it is a path.
    """
    extension = osp.expanduser(extension)
    if osp.exists(extension):
        extension = osp.abspath(extension)
    return extension


def _read_package(target):
    """Read the package data in a given target tarball.
    """
    tar = tarfile.open(target, "r")
    f = tar.extractfile('package/package.json')
    data = json.loads(f.read().decode('utf8'))
    data['jupyterlab_extracted_files'] = [
        f.path[len('package/'):] for f in tar.getmembers()
    ]
    tar.close()
    return data


def _validate_extension(data):
    """Detect if a package is an extension using its metadata.

    Returns any problems it finds.
    """
    jlab = data.get('jupyterlab', None)
    if jlab is None:
        return ['No `jupyterlab` key']
    if not isinstance(jlab, dict):
        return ['The `jupyterlab` key must be a JSON object']
    extension = jlab.get('extension', False)
    mime_extension = jlab.get('mimeExtension', False)
    themeDir = jlab.get('themeDir', '')
    schemaDir = jlab.get('schemaDir', '')

    messages = []
    if not extension and not mime_extension:
        messages.append('No `extension` or `mimeExtension` key present')

    if extension == mime_extension:
        msg = '`mimeExtension` and `extension` must point to different modules'
        messages.append(msg)

    files = data['jupyterlab_extracted_files']
    main = data.get('main', 'index.js')
    if not main.endswith('.js'):
        main += '.js'

    if extension is True:
        extension = main
    elif extension and not extension.endswith('.js'):
        extension += '.js'

    if mime_extension is True:
        mime_extension = main
    elif mime_extension and not mime_extension.endswith('.js'):
        mime_extension += '.js'

    if extension and extension not in files:
        messages.append('Missing extension module "%s"' % extension)

    if mime_extension and mime_extension not in files:
        messages.append('Missing mimeExtension module "%s"' % mime_extension)

    if themeDir and not any(f.startswith(themeDir) for f in files):
        messages.append('themeDir is empty: "%s"' % themeDir)

    if schemaDir and not any(f.startswith(schemaDir) for f in files):
        messages.append('schemaDir is empty: "%s"' % schemaDir)

    return messages


def _tarsum(input_file):
    """
    Compute the recursive sha sum of a tar file.
    """
    tar = tarfile.open(input_file, "r")
    chunk_size = 100 * 1024
    h = hashlib.new("sha1")

    for member in tar:
        if not member.isfile():
            continue
        f = tar.extractfile(member)
        data = f.read(chunk_size)
        while data:
            h.update(data)
            data = f.read(chunk_size)
    return h.hexdigest()


def _get_core_data():
    """Get the data for the app template.
    """
    with open(pjoin(HERE, 'staging', 'package.json')) as fid:
        return json.load(fid)


def _get_static_data(app_dir):
    """Get the data for the app static dir.
    """
    target = pjoin(app_dir, 'static', 'package.json')
    if os.path.exists(target):
        with open(target) as fid:
            return json.load(fid)
    else:
        return None


def _validate_compatibility(extension, deps, core_data):
    """Validate the compatibility of an extension.
    """
    core_deps = core_data['dependencies']
    singletons = core_data['jupyterlab']['singletonPackages']

    errors = []

    for (key, value) in deps.items():
        if key in singletons:
            overlap = _test_overlap(core_deps[key], value)
            if overlap is False:
                errors.append((key, core_deps[key], value))

    return errors


def _test_overlap(spec1, spec2):
    """Test whether two version specs overlap.

    Returns `None` if we cannot determine compatibility,
    otherwise whether there is an overlap
    """
    cmp = _compare_ranges(spec1, spec2)
    if cmp is None:
        return
    return cmp == 0


def _compare_ranges(spec1, spec2):
    """Test whether two version specs overlap.

    Returns `None` if we cannot determine compatibility,
    otherwise return 0 if there is an overlap, 1 if
    spec1 is lower/older than spec2, and -1 if spec1
    is higher/newer than spec2.
    """
    # Test for overlapping semver ranges.
    r1 = Range(spec1, True)
    r2 = Range(spec2, True)

    # If either range is empty, we cannot verify.
    if not r1.range or not r2.range:
        return

    x1 = r1.set[0][0].semver
    x2 = r1.set[0][-1].semver
    y1 = r2.set[0][0].semver
    y2 = r2.set[0][-1].semver

    o1 = r1.set[0][0].operator
    o2 = r2.set[0][0].operator

    # We do not handle (<) specifiers.
    if (o1.startswith('<') or o2.startswith('<')):
        return

    # Handle single value specifiers.
    lx = lte if x1 == x2 else lt
    ly = lte if y1 == y2 else lt
    gx = gte if x1 == x2 else gt
    gy = gte if x1 == x2 else gt

    # Handle unbounded (>) specifiers.
    def noop(x, y, z):
        return True

    if x1 == x2 and o1.startswith('>'):
        lx = noop
    if y1 == y2 and o2.startswith('>'):
        ly = noop

    # Check for overlap.
    if (gte(x1, y1, True) and ly(x1, y2, True) or
        gy(x2, y1, True) and ly(x2, y2, True) or
        gte(y1, x1, True) and lx(y1, x2, True) or
        gx(y2, x1, True) and lx(y2, x2, True)
       ):
       return 0
    if gte(y1, x2, True):
        return 1
    if gte(x1, y2, True):
        return -1
    raise AssertionError('Unexpected case comparing version ranges')


def _is_disabled(name, disabled=[]):
    """Test whether the package is disabled.
    """
    for pattern in disabled:
        if name == pattern:
            return True
        if re.compile(pattern).match(name) is not None:
            return True
    return False


def _format_compatibility_errors(name, version, errors):
    """Format a message for compatibility errors.
    """
    msgs = []
    l0 = 10
    l1 = 10
    for error in errors:
        pkg, jlab, ext = error
        jlab = str(Range(jlab, True))
        ext = str(Range(ext, True))
        msgs.append((pkg, jlab, ext))
        l0 = max(l0, len(pkg) + 1)
        l1 = max(l1, len(jlab) + 1)

    msg = '\n"%s@%s" is not compatible with the current JupyterLab'
    msg = msg % (name, version)
    msg += '\nConflicting Dependencies:\n'
    msg += 'JupyterLab'.ljust(l0)
    msg += 'Extension'.ljust(l1)
    msg += 'Package\n'

    for (pkg, jlab, ext) in msgs:
        msg += jlab.ljust(l0) + ext.ljust(l1) + pkg + '\n'

    return msg


def _get_core_extensions():
    """Get the core extensions.
    """
    data = _get_core_data()['jupyterlab']
    return list(data['extensions']) + list(data['mimeExtensions'])


def _semver_prerelease_key(prerelease):
    """Sort key for prereleases.

    Precedence for two pre-release versions with the same
    major, minor, and patch version MUST be determined by
    comparing each dot separated identifier from left to
    right until a difference is found as follows:
    identifiers consisting of only digits are compare
    numerically and identifiers with letters or hyphens
    are compared lexically in ASCII sort order. Numeric
    identifiers always have lower precedence than non-
    numeric identifiers. A larger set of pre-release
    fields has a higher precedence than a smaller set,
    if all of the preceding identifiers are equal.
    """
    for entry in prerelease:
        if isinstance(entry, int):
            # Assure numerics always sort before string
            yield ('', entry)
        else:
            # Use ASCII compare:
            yield (entry,)


def _semver_key(version, prerelease_first=False):
    """A sort key-function for sorting semver verion string.

    The default sorting order is ascending (0.x -> 1.x -> 2.x).

    If `prerelease_first`, pre-releases will come before
    ALL other semver keys (not just those with same version).
    I.e (1.0-pre, 2.0-pre -> 0.x -> 1.x -> 2.x).

    Otherwise it will sort in the standard way that it simply
    comes before any release with shared version string
    (0.x -> 1.0-pre -> 1.x -> 2.0-pre -> 2.x).
    """
    v = make_semver(version, True)
    if prerelease_first:
        key = (0,) if v.prerelease else (1,)
    else:
        key = ()
    key = key + (v.major, v.minor, v.patch)
    if not prerelease_first:
        #  NOT having a prerelease is > having one
        key = key + (0,) if v.prerelease else (1,)
    if v.prerelease:
        key = key + tuple(_semver_prerelease_key(
            v.prerelease))

    return key


def _fetch_package_metadata(registry, name, logger):
    """Fetch the metadata for a package from the npm registry"""
    req = Request(
        urljoin(registry, quote(name, safe='@')),
        headers={
            'Accept': ('application/vnd.npm.install-v1+json;'
                        ' q=1.0, application/json; q=0.8, */*')
        }
    )
    logger.debug('Fetching URL: %s' % (req.full_url))
    try:
        with urlopen(req) as response:
            return json.load(response)
    except URLError as exc:
        logger.warning(
            'Failed to fetch package metadata for %r: %r',
            name, exc)
        raise


if __name__ == '__main__':
    watch_dev(HERE)

```

```python
# build_handler.py
"""Tornado handlers for frontend config storage."""

# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.
from concurrent.futures import ThreadPoolExecutor
import json
from threading import Event

from notebook.base.handlers import APIHandler
from tornado import gen, web
from tornado.concurrent import run_on_executor

from .commands import build, clean, build_check


class Builder(object):
    building = False
    executor = ThreadPoolExecutor(max_workers=5)
    canceled = False
    _canceling = False
    _kill_event = None
    _future = None

    def __init__(self, log, core_mode, app_dir):
        print('build_handler.Builder###################')

        self.log = log
        self.core_mode = core_mode
        self.app_dir = app_dir

    @gen.coroutine
    def get_status(self):
        print('build_handler.Builder.get_status ###################')

        if self.core_mode:
            raise gen.Return(dict(status='stable', message=''))
        if self.building:
            raise gen.Return(dict(status='building', message=''))

        try:
            messages = yield self._run_build_check(self.app_dir, self.log)
            status = 'needed' if messages else 'stable'
            if messages:
                self.log.warn('Build recommended')
                [self.log.warn(m) for m in messages]
            else:
                self.log.info('Build is up to date')
        except ValueError as e:
            self.log.warn(
                'Could not determine jupyterlab build status without nodejs'
            )
            status = 'stable'
            messages = []

        raise gen.Return(dict(status=status, message='\n'.join(messages)))

    @gen.coroutine
    def build(self):
        print('build_handler.Builder.build###################')

        if self._canceling:
            raise ValueError('Cancel in progress')
        if not self.building:
            self.canceled = False
            self._future = future = gen.Future()
            self.building = True
            self._kill_event = evt = Event()
            try:
                yield self._run_build(self.app_dir, self.log, evt)
                future.set_result(True)
            except Exception as e:
                if str(e) == 'Aborted':
                    future.set_result(False)
                else:
                    future.set_exception(e)
            finally:
                self.building = False
        try:
            yield self._future
        except Exception as e:
            raise e

    @gen.coroutine
    def cancel(self):
        print('build_handler.Builder.cancel###################')

        if not self.building:
            raise ValueError('No current build')
        self._canceling = True
        yield self._future
        self._canceling = False
        self.canceled = True

    @run_on_executor
    def _run_build_check(self, app_dir, logger):
        print('build_handler.Builder._run_build_check###################')

        return build_check(app_dir=app_dir, logger=logger)

    @run_on_executor
    def _run_build(self, app_dir, logger, kill_event):
        print('build_handler.Builder._run_build###################')

        kwargs = dict(app_dir=app_dir, logger=logger, kill_event=kill_event)
        try:
            return build(**kwargs)
        except Exception as e:
            if self._kill_event.is_set():
                return
            self.log.warn('Build failed, running a clean and rebuild')
            clean(app_dir)
            return build(**kwargs)


class BuildHandler(APIHandler):

    def initialize(self, builder):
        print('build_handler.BuildHandler###################')
        self.builder = builder

    @web.authenticated
    @gen.coroutine
    def get(self):
        data = yield self.builder.get_status()
        print(data, '#!#!#!##!###!#!##!#!##!')
        self.finish(json.dumps(data))

    @web.authenticated
    @gen.coroutine
    def delete(self):
        self.log.warn('Canceling build')
        try:
            yield self.builder.cancel()
        except Exception as e:
            raise web.HTTPError(500, str(e))
        self.set_status(204)

    @web.authenticated
    @gen.coroutine
    def post(self):
        self.log.debug('Starting build')
        try:
            yield self.builder.build()
        except Exception as e:
            raise web.HTTPError(500, str(e))

        if self.builder.canceled:
            raise web.HTTPError(400, 'Build canceled')

        self.log.debug('Build succeeded')
        self.set_status(200)


# The path for lab build.
build_path = r"/lab/api/build"
```

```typescript
/**jupyterlab src components SinglePastCommitInfo.tsx*/
import { JupyterLab } from '@jupyterlab/application';

import { SingleCommitInfo, CommitModifiedFile } from '../git';

import { parseFileExtension } from './FileList';

import { ResetDeleteSingleCommit } from './ResetDeleteSingleCommit';

import {
  commitStyle,
  commitNumberLabelStyle,
  commitAuthorLabelStyle,
  commitAuthorIconStyle,
  commitLabelDateStyle,
  commitLabelMessageStyle,
  commitOverviewNumbers,
  commitDetailStyle,
  commitDetailHeader,
  commitDetailFileStyle,
  commitDetailFilePathStyle,
  iconStyle,
  insertionIconStyle,
  numberofChangedFilesStyle,
  deletionIconStyle,
  revertButtonStyle
} from '../componentsStyle/SinglePastCommitInfoStyle';

import {
  changeStageButtonStyle,
  discardFileButtonStyle
} from '../componentsStyle/GitStageStyle';

import { fileIconStyle } from '../componentsStyle/FileItemStyle';

import * as React from 'react';

import { classes } from 'typestyle/';

export interface ISinglePastCommitInfoProps {
  topRepoPath: string;
  num: string;
  data: SingleCommitInfo;
  info: string;
  filesChanged: string;
  insertionCount: string;
  deletionCount: string;
  list: [CommitModifiedFile];
  app: JupyterLab;
  diff: any;
  display: boolean;
  refresh: Function;
  currentTheme: string;
}

export interface ISinglePastCommitInfoState {
  displayDelete: boolean;
  displayReset: boolean;
}

export class SinglePastCommitInfo extends React.Component<
  ISinglePastCommitInfoProps,
  ISinglePastCommitInfoState
> {
  constructor(props: ISinglePastCommitInfoProps) {
    super(props);
    this.state = {
      displayDelete: false,
      displayReset: false
    };
  }

  showDeleteCommit = () => {
    this.setState({
      displayDelete: true,
      displayReset: false
    });
  };

  hideDeleteCommit = () => {
    this.setState({
      displayDelete: false
    });
  };

  showResetToCommit = () => {
    this.setState({
      displayReset: true,
      displayDelete: false
    });
  };

  hideResetToCommit = () => {
    this.setState({
      displayReset: false
    });
  };

  render() {
    return (
      <div>
        <div className={commitStyle}>
          <div>
            <div className={commitAuthorLabelStyle}>
              <span className={commitAuthorIconStyle} />
              {this.props.data.author}
            </div>
            <div className={commitLabelDateStyle}>{this.props.data.date}</div>
            <span className={commitNumberLabelStyle}>
              #{this.props.data.commit
                ? this.props.data.commit.substring(0, 7)
                : ''}
            </span>
          </div>
          <div className={commitLabelMessageStyle}>
            {this.props.data.commit_msg}
          </div>
          <div className={commitOverviewNumbers}>
            <span>
              <span className={classes(iconStyle, numberofChangedFilesStyle)} />
              {this.props.filesChanged}
            </span>
            <span>
              <span
                className={classes(
                  iconStyle,
                  insertionIconStyle(this.props.currentTheme)
                )}
              />
              {this.props.insertionCount}
            </span>
            <span>
              <span
                className={classes(
                  iconStyle,
                  deletionIconStyle(this.props.currentTheme)
                )}
              />
              {this.props.deletionCount}
            </span>
          </div>
        </div>
        <div className={commitDetailStyle}>
          <div className={commitDetailHeader}>
            Changed
            <button
              className={classes(
                changeStageButtonStyle,
                discardFileButtonStyle(this.props.currentTheme)
              )}
              onClick={this.showDeleteCommit}
            />
            <button
              className={classes(
                changeStageButtonStyle,
                revertButtonStyle(this.props.currentTheme)
              )}
              onClick={this.showResetToCommit}
            />
          </div>
          <div>
            {this.state.displayDelete && (
              <ResetDeleteSingleCommit
                action="delete"
                commitId={this.props.data.commit}
                path={this.props.topRepoPath}
                onCancel={this.hideDeleteCommit}
                refresh={this.props.refresh}
              />
            )}
            {this.state.displayReset && (
              <ResetDeleteSingleCommit
                action="reset"
                commitId={this.props.data.commit}
                path={this.props.topRepoPath}
                onCancel={this.hideResetToCommit}
                refresh={this.props.refresh}
              />
            )}
          </div>
          {this.props.list.length > 0 &&
            this.props.list.map((modifiedFile, modifiedFileIndex) => {
              return (
                <li className={commitDetailFileStyle} key={modifiedFileIndex}>
                  <span
                    className={`${fileIconStyle} ${parseFileExtension(
                      modifiedFile.modified_file_path
                    )}`}
                    onDoubleClick={() => {
                      window.open(
                        'https://github.com/search?q=' +
                          this.props.data.commit +
                          '&type=Commits&utf8=%E2%9C%93'
                      );
                    }}
                  />
                  <span
                    className={commitDetailFilePathStyle}
                    onDoubleClick={() => {
                      this.props.diff(
                        this.props.app,
                        modifiedFile.modified_file_path,
                        this.props.data.commit,
                        this.props.data.pre_commit
                      );
                    }}
                  >
                    {modifiedFile.modified_file_name}
                  </span>
                </li>
              );
            })}
        </div>
      </div>
    );
  }
}
```

```typescript
/** Jupyterlab-git/src/components/ResetDeleteSingleCommit.tsx*/
import * as React from 'react';

import { classes } from 'typestyle';

import {
  WarningLabel,
  MessageInput,
  Button,
  ResetDeleteButton,
  CancelButton,
  ResetDeleteDisabledButton
} from '../componentsStyle/SinglePastCommitInfoStyle';

import { Git } from '../git';

export interface IResetDeleteProps {
  action: 'reset' | 'delete';
  commitId: string;
  path: string;
  onCancel: Function;
  refresh: Function;
}

export interface IResetDeleteState {
  message: string;
  resetDeleteDisabled: boolean;
}

export class ResetDeleteSingleCommit extends React.Component<
  IResetDeleteProps,
  IResetDeleteState
> {
  constructor(props: IResetDeleteProps) {
    super(props);
    this.state = {
      message: '',
      resetDeleteDisabled: true
    };
  }

  onCancel = () => {
    this.setState({
      message: '',
      resetDeleteDisabled: true
    });
    this.props.onCancel();
  };

  updateMessage = (value: string) => {
    this.setState({
      message: value,
      resetDeleteDisabled: value === ''
    });
  };

  handleResetDelete = () => {
    let gitApi = new Git();
    if (this.props.action === 'reset') {
      gitApi
        .resetToCommit(this.state.message, this.props.path, this.props.commitId)
        .then(response => {
          this.props.refresh();
        });
    } else {
      gitApi
        .deleteCommit(this.state.message, this.props.path, this.props.commitId)
        .then(response => {
          this.props.refresh();
        });
    }
    this.props.onCancel();
  };

  render() {
    return (
      <div>
        <div className={WarningLabel}>
          {this.props.action === 'delete'
            ? "These changes will be gone forever. Commit if you're sure?"
            : "All changes after this will be gone forever. Commit if you're sure?"}
        </div>
        <input
          type="text"
          className={MessageInput}
          onChange={event => this.updateMessage(event.currentTarget.value)}
          placeholder="Add a commit message to make changes"
        />
        <button
          className={classes(Button, CancelButton)}
          onClick={this.onCancel}
        >
          Cancel
        </button>
        <button
          className={
            this.state.resetDeleteDisabled
              ? classes(Button, ResetDeleteDisabledButton)
              : classes(Button, ResetDeleteButton)
          }
          disabled={this.state.resetDeleteDisabled}
          onClick={this.handleResetDelete}
        >
          {this.props.action === 'delete'
            ? 'Delete these changes'
            : 'Revert to this commit'}
        </button>
      </div>
    );
  }
}
```

```typescript
/** jupyterlab-git/src/components/PathHeader.tsx */
import {
  repoStyle,
  repoPathStyle,
  repoRefreshStyle,
} from '../componentsStyle/PathHeaderStyle';

import * as React from 'react';

import { classes } from 'typestyle';

export interface IPathHeaderState {
  topRepoPath: string;
  refresh: any;
}

export interface IPathHeaderProps {
  currentFileBrowserPath: string;
  topRepoPath: string;
  refresh: any;
  currentBranch: string;
}

export class PathHeader extends React.Component<
  IPathHeaderProps,
  IPathHeaderState
> {
  constructor(props: IPathHeaderProps) {
    super(props);
    this.state = {
      topRepoPath: props.topRepoPath,
      refresh: props.refresh
    };
  }

  render() {
    let relativePath = this.props.currentFileBrowserPath.split('/');
    return (
      <div className={repoStyle}>
        <span className={repoPathStyle}>
          {relativePath[relativePath.length - 1]+" / "+this.props.currentBranch}
        </span>
        <button
          className={classes(repoRefreshStyle, 'jp-Icon-16')}
          onClick={() => this.props.refresh()}
        />
      </div>
    );
  }
}
```

```typescript
/** jupyter-git/src/components/PastCommits.tsx*/
import { JupyterLab } from '@jupyterlab/application';

import { FileList } from './FileList';

import { SinglePastCommitInfo } from './SinglePastCommitInfo';

import { pastCommitsContainerStyle } from '../componentsStyle/PastCommitsStyle';

import * as React from 'react';

/** Interface for PastCommits component props */
export interface IPastCommitsProps {
  currentFileBrowserPath: string;
  topRepoPath: string;
  pastCommits: any;
  inNewRepo: boolean;
  showList: boolean;
  stagedFiles: any;
  unstagedFiles: any;
  untrackedFiles: any;
  app: JupyterLab;
  refresh: any;
  diff: any;
  pastCommitInfo: string;
  pastCommitFilesChanged: string;
  pastCommitInsertionCount: string;
  pastCommitDeletionCount: string;
  pastCommitData: any;
  pastCommitNumber: any;
  pastCommitFilelist: any;
  sideBarExpanded: boolean;
  currentTheme: string;
}

export class PastCommits extends React.Component<IPastCommitsProps, {}> {
  constructor(props: IPastCommitsProps) {
    super(props);
  }

  render() {
    return (
      <div className={pastCommitsContainerStyle}>
        {!this.props.showList && (
          <SinglePastCommitInfo
            topRepoPath={this.props.topRepoPath}
            num={this.props.pastCommitNumber}
            data={this.props.pastCommitData}
            info={this.props.pastCommitInfo}
            filesChanged={this.props.pastCommitFilesChanged}
            insertionCount={this.props.pastCommitInsertionCount}
            deletionCount={this.props.pastCommitDeletionCount}
            list={this.props.pastCommitFilelist}
            app={this.props.app}
            diff={this.props.diff}
            display={!this.props.showList}
            currentTheme={this.props.currentTheme}
            refresh={this.props.refresh}
          />
        )}
        <FileList
          currentFileBrowserPath={this.props.currentFileBrowserPath}
          topRepoPath={this.props.topRepoPath}
          stagedFiles={this.props.stagedFiles}
          unstagedFiles={this.props.unstagedFiles}
          untrackedFiles={this.props.untrackedFiles}
          app={this.props.app}
          refresh={this.props.refresh}
          sideBarExpanded={this.props.sideBarExpanded}
          display={this.props.showList}
          currentTheme={this.props.currentTheme}
        />
      </div>
    );
  }
}
```

```typescript
/** jupyterlab-git/src/components/PastCommitNode.tsx*/
import {
  pastCommitNodeStyle,
  pastCommitWorkingNodeStyle,
  pastCommitContentStyle,
  pastCommitWorkingContentStyle,
  pastCommitHeadContentStyle,
  pastCommitNumberContentStyle,
  pastCommitActiveContentStyle,
  pastCommitLineStyle
} from '../componentsStyle/PastCommitNodeStyle';

import { classes } from 'typestyle';

import * as React from 'react';

export interface IPastCommitNodeProps {
  index: number;
  isLast: boolean;
  pastCommit: any;
  currentFileBrowserPath: string;
  setShowList: Function;
  getPastCommit: Function;
  activeNode: number;
  updateActiveNode: Function;
  isVisible: boolean;
}

export class PastCommitNode extends React.Component<IPastCommitNodeProps, {}> {
  constructor(props: IPastCommitNodeProps) {
    super(props);
  }

  getPastCommitNodeClass(): string {
    if (!this.props.isVisible) {
      return null;
    }
    return this.props.index === -1
      ? classes(pastCommitWorkingNodeStyle, pastCommitNodeStyle)
      : classes(pastCommitNodeStyle);
  }

  getPastCommitLineClass(): string {
    if (!this.props.isVisible) {
      return null;
    }
    return this.props.isLast ? null : classes(pastCommitLineStyle);
  }

  getContent(): string | number {
    if (this.props.index === -1) {
      return 'WORKING';
    } else if (this.props.index === 0) {
      return 'HEAD';
    } else {
      return this.props.index;
    }
  }

  getContentClass(): string {
    const activeContentStyle =
      this.props.index === this.props.activeNode
        ? pastCommitActiveContentStyle
        : null;
    if (!this.props.isVisible) {
      return null;
    }
    if (this.props.index === -1) {
      return classes(
        pastCommitWorkingContentStyle,
        pastCommitContentStyle,
        activeContentStyle
      );
    } else if (this.props.index === 0) {
      return classes(
        pastCommitHeadContentStyle,
        pastCommitContentStyle,
        activeContentStyle
      );
    } else {
      return classes(
        pastCommitNumberContentStyle,
        pastCommitContentStyle,
        activeContentStyle
      );
    }
  }

  handleClick(): void {
    this.props.index === -1
      ? this.props.setShowList(true)
      : (this.props.getPastCommit(
          this.props.pastCommit,
          this.props.index,
          this.props.currentFileBrowserPath
        ),
        this.props.setShowList(false));
    this.props.updateActiveNode(this.props.index);
  }

  render() {
    return (
      <div key={this.props.index}>
        <div
          className={this.getPastCommitNodeClass()}
          onClick={() => this.handleClick()}
        >
          <span className={this.getContentClass()}>
            {this.props.isVisible && this.getContent()}
          </span>
        </div>
        <div className={this.getPastCommitLineClass()} />
      </div>
    );
  }
}
```

```typescript
/** jupyterlab-git/src/components/NewBranchBox.tsx*/
import * as React from 'react';

import {
  newBranchBoxStyle,
  newBranchInputAreaStyle,
  buttonStyle,
  newBranchButtonStyle,
  cancelNewBranchButtonStyle
} from '../componentsStyle/NewBranchBoxStyle';

import { classes } from 'typestyle';

export interface ICommitBoxProps {
  createNewBranch: Function;
  toggleNewBranchBox: Function;
}

export interface ICommitBoxState {
  value: string;
}

export class NewBranchBox extends React.Component<
  ICommitBoxProps,
  ICommitBoxState
> {
  constructor(props: ICommitBoxProps) {
    super(props);
    this.state = {
      value: ''
    };
  }

  /** Prevent enter key triggered 'submit' action during input */
  onKeyPress(event: any): void {
    if (event.which === 13) {
      event.preventDefault();
      this.setState({ value: this.state.value + '\n' });
    }
  }

  /** Handle input inside commit message box */
  handleChange = (event: any): void => {
    this.setState({
      value: event.target.value
    });
  };

  render() {
    return (
      <div
        className={newBranchInputAreaStyle}
        onKeyPress={event => this.onKeyPress(event)}
      >
        <input
          className={newBranchBoxStyle}
          placeholder={'New branch'}
          value={this.state.value}
          onChange={this.handleChange}
          autoFocus
        />
        <input
          className={classes(buttonStyle, newBranchButtonStyle)}
          type="button"
          title="Create New"
          onClick={() => this.props.createNewBranch(this.state.value)}
        />
        <input
          className={classes(buttonStyle, cancelNewBranchButtonStyle)}
          type="button"
          title="Cancel"
          onClick={() => this.props.toggleNewBranchBox()}
        />
      </div>
    );
  }
}
```

```typescript
/** jupyterlab-git/src/components/HistorySideBar.tsx*/
import { PastCommitNode } from './PastCommitNode';

import { SingleCommitInfo } from '../git';

import {
  historySideBarStyle,
  historySideBarExpandedStyle
} from '../componentsStyle/HistorySideBarStyle';

import { classes } from 'typestyle';

import * as React from 'react';

/** Interface for PastCommits component props */
export interface IHistorySideBarProps {
  currentFileBrowserPath: string;
  pastCommits: SingleCommitInfo[];
  isExpanded: boolean;
  setShowList: Function;
  getPastCommit: Function;
}

/** Interface for PastCommits component state */
export interface IHistorySideBarState {
  activeNode: number;
}

export class HistorySideBar extends React.Component<
  IHistorySideBarProps,
  IHistorySideBarState
> {
  constructor(props: IHistorySideBarProps) {
    super(props);

    this.state = {
      activeNode: -1
    };
  }

  getSideBarClass(): string {
    return this.props.isExpanded
      ? classes(historySideBarExpandedStyle, historySideBarStyle)
      : historySideBarStyle;
  }

  updateActiveNode = (index: number): void => {
    this.setState({ activeNode: index });
  };

  render() {
    return (
      <div className={this.getSideBarClass()}>
        <PastCommitNode
          key={-1}
          index={-1}
          isLast={false}
          pastCommit={null}
          currentFileBrowserPath={this.props.currentFileBrowserPath}
          setShowList={this.props.setShowList}
          getPastCommit={this.props.getPastCommit}
          activeNode={this.state.activeNode}
          updateActiveNode={this.updateActiveNode}
          isVisible={this.props.isExpanded}
        />
        {this.props.pastCommits.map(
          (pastCommit: SingleCommitInfo, pastCommitIndex: number) => (
            <PastCommitNode
              key={pastCommitIndex}
              index={pastCommitIndex}
              isLast={pastCommitIndex === this.props.pastCommits.length - 1}
              pastCommit={pastCommit}
              currentFileBrowserPath={this.props.currentFileBrowserPath}
              setShowList={this.props.setShowList}
              getPastCommit={this.props.getPastCommit}
              activeNode={this.state.activeNode}
              updateActiveNode={this.updateActiveNode}
              isVisible={this.props.isExpanded}
            />
          )
        )}
      </div>
    );
  }
}
```

```typescript
/** jupyterlab-git/src/components/GitWidget.tsx */
import * as React from 'react';

import * as ReactDOM from 'react-dom';

import { ServiceManager } from '@jupyterlab/services';

import { Message } from '@phosphor/messaging';

import { Widget } from '@phosphor/widgets';

import { JupyterLab } from '@jupyterlab/application';

import { ISignal, Signal } from '@phosphor/signaling';

import { GitPanel } from './GitPanel';

import { gitWidgetStyle } from '../componentsStyle/GitWidgetStyle';

import '../../style/variables.css';

/**
 * An options object for creating a running sessions widget.
 */
export interface IOptions {
  /**
   * A service manager instance.
   */
  manager: ServiceManager.IManager;

  /**
   * The renderer for the running sessions widget.
   * The default is a shared renderer instance.
   */
  renderer?: IRenderer;
}

/**
 * A renderer for use with a running sessions widget.
 */
export interface IRenderer {
  createNode(): HTMLElement;
}

/**
 * The default implementation of `IRenderer`.
 */
export class Renderer implements IRenderer {
  createNode(): HTMLElement {
    let node = document.createElement('div');
    node.id = 'GitSession-root';

    return node;
  }
}

/**
 * The default `Renderer` instance.
 */
export const defaultRenderer = new Renderer();

/**
 * A class that exposes the git-plugin sessions.
 */
export class GitWidget extends Widget {
  component: any;
  /**
   * Construct a new running widget.
   */
  constructor(app: JupyterLab, options: IOptions, diff_function: any) {
    super({
      node: (options.renderer || defaultRenderer).createNode()
    });
    this.addClass(gitWidgetStyle);
    const element = <GitPanel app={app} diff={diff_function} />;
    this.component = ReactDOM.render(element, this.node);
    this.component.refresh();
  }

  /**
   * Override widget's default show() to 
   * refresh content every time Git widget is shown.
   */
  show(): void {
    super.show();
    this.component.refresh();
  }

  /**
   * The renderer used by the running sessions widget.
   */
  get renderer(): IRenderer {
    return this._renderer;
  }

  /**
   * A signal emitted when the directory listing is refreshed.
   */
  get refreshed(): ISignal<this, void> {
    return this._refreshed;
  }

  /**
   * Get the input text node.
   */
  get inputNode(): HTMLInputElement {
    return this.node.getElementsByTagName('input')[0] as HTMLInputElement;
  }

  /**
   * Dispose of the resources used by the widget.
   */
  dispose(): void {
    this._renderer = null;
    clearTimeout(this._refreshId);
    super.dispose();
  }

  /**
   * Handle the DOM events for the widget.
   *
   * @param event - The DOM event sent to the widget.
   *
   * #### Notes
   * This method implements the DOM `EventListener` interface and is
   * called in response to events on the widget's DOM nodes. It should
   * not be called directly by user code.
   */
  handleEvent(event: Event): void {
    switch (event.type) {
      case 'change':
        this._evtChange(event as MouseEvent);
      case 'click':
        this._evtClick(event as MouseEvent);
        break;
      case 'dblclick':
        this._evtDblClick(event as MouseEvent);
        break;
      default:
        break;
    }
  }

  /**
   * A message handler invoked on an `'after-attach'` message.
   */
  protected onAfterAttach(msg: Message): void {
    this.node.addEventListener('change', this);
    this.node.addEventListener('click', this);
    this.node.addEventListener('dblclick', this);
  }

  /**
   * A message handler invoked on a `'before-detach'` message.
   */
  protected onBeforeDetach(msg: Message): void {
    this.node.addEventListener('change', this);
    this.node.removeEventListener('click', this);
    this.node.removeEventListener('dblclick', this);
  }

  /**
   * Handle the `'click'` event for the widget.
   *
   * #### Notes
   * This listener is attached to the document node.
   */
  private _evtChange(event: MouseEvent): void {}
  /**
   * Handle the `'click'` event for the widget.
   *
   * #### Notes
   * This listener is attached to the document node.
   */
  private _evtClick(event: MouseEvent): void {}

  /**
   * Handle the `'dblclick'` event for the widget.
   */
  private _evtDblClick(event: MouseEvent): void {}

  private _renderer: IRenderer = null;
  private _refreshId = -1;
  private _refreshed = new Signal<this, void>(this);
}
```

```typescript
/** jupyterlab-git/src/components/GitStage.tsx */
import { JupyterLab } from '@jupyterlab/application';

import { GitStatusFileResult } from '../git';

import {
  sectionFileContainerStyle,
  sectionFileContainerDisabledStyle,
  sectionAreaStyle,
  sectionHeaderLabelStyle,
  changeStageButtonStyle,
  caretdownImageStyle,
  caretrightImageStyle,
  changeStageButtonLeftStyle,
  discardFileButtonStyle,
  discardAllWarningStyle
} from '../componentsStyle/GitStageStyle';

import {
  cancelDiscardButtonStyle,
  acceptDiscardButtonStyle,
  discardButtonStyle,
  discardWarningStyle
} from '../componentsStyle/FileItemStyle';

import { FileItem } from './FileItem';

import { classes } from 'typestyle';

import * as React from 'react';

export interface IGitStageProps {
  heading: string;
  topRepoPath: string;
  files: any;
  app: JupyterLab;
  refresh: any;
  showFiles: boolean;
  displayFiles: Function;
  moveAllFiles: Function;
  discardAllFiles: Function;
  discardFile: Function;
  moveFile: Function;
  moveFileIconClass: Function;
  moveFileIconSelectedClass: string;
  moveAllFilesTitle: string;
  moveFileTitle: string;
  openFile: Function;
  extractFilename: Function;
  contextMenu: Function;
  parseFileExtension: Function;
  parseSelectedFileExtension: Function;
  selectedFile: number;
  updateSelectedFile: Function;
  selectedStage: string;
  selectedDiscardFile: number;
  updateSelectedDiscardFile: Function;
  disableFiles: boolean;
  toggleDisableFiles: Function;
  updateSelectedStage: Function;
  isDisabled: boolean;
  disableOthers: Function;
  sideBarExpanded: boolean;
  currentTheme: string;
}

export interface IGitStageState {
  showDiscardWarning: boolean;
}

export class GitStage extends React.Component<IGitStageProps, IGitStageState> {
  constructor(props: IGitStageProps) {
    super(props);
    this.state = {
      showDiscardWarning: false
    };
  }

  checkContents() {
    if (this.props.files.length > 0) {
      return false;
    } else {
      return true;
    }
  }

  checkDisabled() {
    return this.props.isDisabled
      ? classes(sectionFileContainerStyle, sectionFileContainerDisabledStyle)
      : sectionFileContainerStyle;
  }

  toggleDiscardChanges() {
    this.setState({ showDiscardWarning: !this.state.showDiscardWarning }, () =>
      this.props.disableOthers()
    );
  }

  render() {
    return (
      <div className={this.checkDisabled()}>
        <div className={sectionAreaStyle}>
          <span className={sectionHeaderLabelStyle}>
            {this.props.heading}({this.props.files.length})
          </span>
          {this.props.files.length > 0 && (
            <button
              className={
                this.props.showFiles
                  ? `${changeStageButtonStyle} ${caretdownImageStyle}`
                  : `${changeStageButtonStyle} ${caretrightImageStyle}`
              }
              onClick={() => this.props.displayFiles()}
            />
          )}
          <button
            disabled={this.checkContents()}
            className={`${this.props.moveFileIconClass(
              this.props.currentTheme
            )} ${changeStageButtonStyle} 
               ${changeStageButtonLeftStyle}
            title={this.props.moveAllFilesTitle}
            onClick={() =>
              this.props.moveAllFiles(
                this.props.topRepoPath,
                this.props.refresh
              )}
          />
          {this.props.heading === 'Changed' && (
            <button
              disabled={this.checkContents()}
              className={classes(
                changeStageButtonStyle,
                discardFileButtonStyle(this.props.currentTheme)
              )}
              title={'Discard All Changes'}
              onClick={() => this.toggleDiscardChanges()}
            />
          )}
        </div>
        {this.props.showFiles && (
          <div className={sectionFileContainerStyle}>
            {this.state.showDiscardWarning && (
              <div
                className={classes(discardAllWarningStyle, discardWarningStyle)}
              >
                These changes will be gone forever
                <div>
                  <button
                    className={classes(
                      discardButtonStyle,
                      cancelDiscardButtonStyle
                    )}
                    onClick={() => this.toggleDiscardChanges()}
                  >
                    Cancel
                  </button>
                  <button
                    className={classes(
                      discardButtonStyle,
                      acceptDiscardButtonStyle
                    )}
                    onClick={() => {
                      this.props.discardAllFiles(
                        this.props.topRepoPath,
                        this.props.refresh
                      );
                      this.toggleDiscardChanges();
                    }}
                  >
                    Discard
                  </button>
                </div>
              </div>
            )}
            {this.props.files.map(
              (file: GitStatusFileResult, file_index: number) => {
                return (
                  <FileItem
                    key={file_index}
                    topRepoPath={this.props.topRepoPath}
                    stage={this.props.heading}
                    file={file}
                    app={this.props.app}
                    refresh={this.props.refresh}
                    moveFile={this.props.moveFile}
                    discardFile={this.props.discardFile}
                    moveFileIconClass={this.props.moveFileIconClass}
                    moveFileIconSelectedClass={
                      this.props.moveFileIconSelectedClass
                    }
                    moveFileTitle={this.props.moveFileTitle}
                    openFile={this.props.openFile}
                    extractFilename={this.props.extractFilename}
                    contextMenu={this.props.contextMenu}
                    parseFileExtension={this.props.parseFileExtension}
                    parseSelectedFileExtension={
                      this.props.parseSelectedFileExtension
                    }
                    selectedFile={this.props.selectedFile}
                    updateSelectedFile={this.props.updateSelectedFile}
                    fileIndex={file_index}
                    selectedStage={this.props.selectedStage}
                    selectedDiscardFile={this.props.selectedDiscardFile}
                    updateSelectedDiscardFile={
                      this.props.updateSelectedDiscardFile
                    }
                    disableFile={this.props.disableFiles}
                    toggleDisableFiles={this.props.toggleDisableFiles}
                    sideBarExpanded={this.props.sideBarExpanded}
                    currentTheme={this.props.currentTheme}
                  />
                );
              }
            )}
          </div>
        )}
      </div>
    );
  }
}
```

```typescript
/** jupyterlab-git/src/components/GitPanel.tsx */
import * as React from 'react';

import { JupyterLab } from '@jupyterlab/application';

import {
  Git,
  GitBranchResult,
  GitStatusResult,
  GitShowTopLevelResult,
  GitAllHistory,
  GitLogResult,
  SingleCommitInfo,
  GitStatusFileResult
} from '../git';

import { PathHeader } from './PathHeader';

import { BranchHeader } from './BranchHeader';

import { PastCommits } from './PastCommits';

import { HistorySideBar } from './HistorySideBar';

import {
  panelContainerStyle,
  panelPushedContentStyle,
  panelContentStyle,
  panelWarningStyle,
  findRepoButtonStyle
} from '../componentsStyle/GitPanelStyle';

import { classes } from 'typestyle';

/** Interface for GitPanel component state */
export interface IGitSessionNodeState {
  currentFileBrowserPath: string;
  topRepoPath: string;
  showWarning: boolean;

  branches: any;
  currentBranch: string;
  disableSwitchBranch: boolean;

  pastCommits: any;
  inNewRepo: boolean;
  showList: boolean;

  stagedFiles: any;
  unstagedFiles: any;
  untrackedFiles: any;

  sideBarExpanded: boolean;

  pastCommitInfo: string;
  pastCommitFilesChanged: string;
  pastCommitInsertionCount: string;
  pastCommitDeletionCount: string;
  pastCommitData: any;
  pastCommitNumber: any;
  pastCommitFilelist: any;
}

/** Interface for GitPanel component props */
export interface IGitSessionNodeProps {
  app: JupyterLab;
  diff: any;
}

/** A React component for the git extension's main display */
export class GitPanel extends React.Component<
  IGitSessionNodeProps,
  IGitSessionNodeState
> {
  constructor(props: IGitSessionNodeProps) {
    super(props);
    this.state = {
      currentFileBrowserPath: '',
      topRepoPath: '',
      showWarning: false,
      branches: [],
      currentBranch: '',
      disableSwitchBranch: true,
      pastCommits: [],
      inNewRepo: true,
      showList: true,
      stagedFiles: [],
      unstagedFiles: [],
      untrackedFiles: [],
      sideBarExpanded: false,
      pastCommitInfo: '',
      pastCommitFilesChanged: '',
      pastCommitInsertionCount: '',
      pastCommitDeletionCount: '',
      pastCommitData: '',
      pastCommitNumber: '',
      pastCommitFilelist: ''
    };
  }

  setShowList = (state: boolean) => {
    this.setState({ showList: state });
  };

  /** Show the commit message and changes from a past commit */
  showPastCommitWork = async (
    pastCommit: SingleCommitInfo,
    pastCommitIndex: number,
    path: string
  ) => {
    let gitApi = new Git();
    let detailedLogData = await gitApi.detailedLog(pastCommit.commit, path);
    if (detailedLogData.code === 0) {
      this.setState({
        pastCommitInfo: detailedLogData.modified_file_note,
        pastCommitFilesChanged: detailedLogData.modified_files_count,
        pastCommitInsertionCount: detailedLogData.number_of_insertions,
        pastCommitDeletionCount: detailedLogData.number_of_deletions,
        pastCommitData: pastCommit,
        pastCommitNumber: pastCommitIndex + ' commit(s) before',
        pastCommitFilelist: detailedLogData.modified_files
      });
    }
  };

  /** 
   * Refresh widget, update all content 
   */
  refresh = async () => {
    try {
      let leftSidebarItems = this.props.app.shell.widgets('left');
      let fileBrowser = leftSidebarItems.next();
      while (fileBrowser && fileBrowser.id !== 'filebrowser') {
        fileBrowser = leftSidebarItems.next();
      }
      let gitApi = new Git();
      // If fileBrowser has loaded, make API request
      if (fileBrowser) {
        // Make API call to get all git info for repo
        let apiResult = await gitApi.allHistory(
          (fileBrowser as any).model.path
        );

        if (apiResult.code === 0) {
          // Get top level path of repo
          let apiShowTopLevel = (apiResult as GitAllHistory).data
            .show_top_level;

          // Get current git branch
          let branchData = (apiResult as GitAllHistory).data.branch;
          let currentBranch = 'master';
          if (branchData.code === 0) {
            let allBranches = (branchData as GitBranchResult).branches;
            for (var i = 0; i < allBranches.length; i++) {
              if (allBranches[i].current[0]) {
                currentBranch = allBranches[i].name;
                break;
              }
            }
          }

          // Get git log for current branch
          let logData = (apiResult as GitAllHistory).data.log;
          let pastCommits = new Array<SingleCommitInfo>();
          if (logData.code === 0) {
            pastCommits = (logData as GitLogResult).commits;
          }

          // Get git status for current branch
          let stagedFiles = new Array<GitStatusFileResult>(),
            unstagedFiles = new Array<GitStatusFileResult>(),
            untrackedFiles = new Array<GitStatusFileResult>();
          let changedFiles = 0;
          let disableSwitchBranch = true;
          let statusData = (apiResult as GitAllHistory).data.status;
          if (statusData.code === 0) {
            let statusFiles = (statusData as GitStatusResult).files;
            for (let i = 0; i < statusFiles.length; i++) {
              // If file has been changed
              if (statusFiles[i].x !== '?' && statusFiles[i].x !== '!') {
                changedFiles++;
              }
              // If file is untracked
              if (statusFiles[i].x === '?' && statusFiles[i].y === '?') {
                untrackedFiles.push(statusFiles[i]);
              } else {
                // If file is staged
                if (statusFiles[i].x !== ' ' && statusFiles[i].y !== 'D') {
                  stagedFiles.push(statusFiles[i]);
                }
                // If file is unstaged but tracked
                if (statusFiles[i].y !== ' ') {
                  unstagedFiles.push(statusFiles[i]);
                }
              }
            }
            // No uncommitted changed files, allow switching branches
            if (changedFiles === 0) {
              disableSwitchBranch = false;
            }
          }
          // No committed files ever, disable switching branches
          if (pastCommits.length === 0) {
            disableSwitchBranch = true;
          }

          // If not in same repo as before refresh, display the current repo
          let inNewRepo =
            this.state.topRepoPath !==
            (apiShowTopLevel as GitShowTopLevelResult).top_repo_path;
          let showList = this.state.showList;
          if (inNewRepo) {
            showList = true;
          }

          this.setState({
            currentFileBrowserPath: (fileBrowser as any).model.path,
            topRepoPath: (apiShowTopLevel as GitShowTopLevelResult)
              .top_repo_path,
            showWarning: true,
            branches: (branchData as GitBranchResult).branches,
            currentBranch: currentBranch,
            disableSwitchBranch: disableSwitchBranch,
            pastCommits: pastCommits,
            inNewRepo: inNewRepo,
            showList: showList,
            stagedFiles: stagedFiles,
            unstagedFiles: unstagedFiles,
            untrackedFiles: untrackedFiles
          });
        } else {
          this.setState({
            currentFileBrowserPath: (fileBrowser as any).model.path,
            topRepoPath: '',
            showWarning: false
          });
        }
      }
    } catch (err) {
      console.log(err);
    }
  };

  toggleSidebar = (): void => {
    this.setState({ sideBarExpanded: !this.state.sideBarExpanded });
  };

  getContentClass(): string {
    return this.state.sideBarExpanded
      ? classes(panelPushedContentStyle, panelContentStyle)
      : panelContentStyle;
  }

  render() {
    return (
      <div className={panelContainerStyle}>
        <PathHeader
          currentFileBrowserPath={this.state.currentFileBrowserPath}
          topRepoPath={this.state.topRepoPath}
          refresh={this.refresh}
          currentBranch={this.state.currentBranch}
        />
        <div className={this.getContentClass()}>
          {this.state.showWarning && (
            <div>
              <HistorySideBar
                isExpanded={this.state.sideBarExpanded}
                currentFileBrowserPath={this.state.currentFileBrowserPath}
                pastCommits={this.state.pastCommits}
                setShowList={this.setShowList}
                getPastCommit={this.showPastCommitWork}
              />
              <BranchHeader
                currentFileBrowserPath={this.state.currentFileBrowserPath}
                topRepoPath={this.state.topRepoPath}
                refresh={this.refresh}
                currentBranch={this.state.currentBranch}
                stagedFiles={this.state.stagedFiles}
                data={this.state.branches}
                disabled={this.state.disableSwitchBranch}
                toggleSidebar={this.toggleSidebar}
                showList={this.state.showList}
                currentTheme={this.props.app.shell.dataset.themeLight}
              />
              <PastCommits
                currentFileBrowserPath={this.state.currentFileBrowserPath}
                topRepoPath={this.state.topRepoPath}
                pastCommits={this.state.pastCommits}
                inNewRepo={this.state.inNewRepo}
                showList={this.state.showList}
                stagedFiles={this.state.stagedFiles}
                unstagedFiles={this.state.unstagedFiles}
                untrackedFiles={this.state.untrackedFiles}
                app={this.props.app}
                refresh={this.refresh}
                diff={this.props.diff}
                pastCommitInfo={this.state.pastCommitInfo}
                pastCommitFilesChanged={this.state.pastCommitFilesChanged}
                pastCommitInsertionCount={this.state.pastCommitInsertionCount}
                pastCommitDeletionCount={this.state.pastCommitDeletionCount}
                pastCommitData={this.state.pastCommitData}
                pastCommitNumber={this.state.pastCommitNumber}
                pastCommitFilelist={this.state.pastCommitFilelist}
                sideBarExpanded={this.state.sideBarExpanded}
                currentTheme={this.props.app.shell.dataset.themeLight}
              />
            </div>
          )}
          {!this.state.showWarning && (
            <div className={panelWarningStyle}>
              <div>You aren’t in a git repository.</div>
              <button
                className={findRepoButtonStyle}
                onClick={() =>
                  this.props.app.commands.execute('filebrowser:toggle-main')}
              >
                Go find a repo
              </button>
            </div>
          )}
        </div>
      </div>
    );
  }
}

```

```typescript
/** jupyterlab-git/src/components/FileList.tsx */
import { Dialog, showDialog } from '@jupyterlab/apputils';

import { JupyterLab } from '@jupyterlab/application';

import { Menu } from '@phosphor/widgets';

import { PathExt } from '@jupyterlab/coreutils';

import { Git, GitShowPrefixResult } from '../git';

import {
  moveFileUpButtonStyle,
  moveFileDownButtonStyle,
  moveFileUpButtonSelectedStyle,
  moveFileDownButtonSelectedStyle,
  folderFileIconStyle,
  genericFileIconStyle,
  yamlFileIconStyle,
  markdownFileIconStyle,
  imageFileIconStyle,
  spreadsheetFileIconStyle,
  jsonFileIconStyle,
  pythonFileIconStyle,
  kernelFileIconStyle,
  folderFileIconSelectedStyle,
  genericFileIconSelectedStyle,
  yamlFileIconSelectedStyle,
  markdownFileIconSelectedStyle,
  imageFileIconSelectedStyle,
  spreadsheetFileIconSelectedStyle,
  jsonFileIconSelectedStyle,
  pythonFileIconSelectedStyle,
  kernelFileIconSelectedStyle
} from '../componentsStyle/FileListStyle';

import { GitStage } from './GitStage';

import * as React from 'react';

export namespace CommandIDs {
  export const gitFileOpen = 'gf:Open';
  export const gitFileUnstage = 'gf:Unstage';
  export const gitFileStage = 'gf:Stage';
  export const gitFileTrack = 'gf:Track';
  export const gitFileUntrack = 'gf:Untrack';
  export const gitFileDiscard = 'gf:Discard';
}

export interface IFileListState {
  commitMessage: string;
  disableCommit: boolean;
  showStaged: boolean;
  showUnstaged: boolean;
  showUntracked: boolean;
  contextMenuStaged: any;
  contextMenuUnstaged: any;
  contextMenuUntracked: any;
  contextMenuTypeX: string;
  contextMenuTypeY: string;
  contextMenuFile: string;
  contextMenuIndex: number;
  contextMenuStage: string;
  selectedFile: number;
  selectedStage: string;
  selectedDiscardFile: number;
  disableStaged: boolean;
  disableUnstaged: boolean;
  disableUntracked: boolean;
  disableFiles: boolean;
}

export interface IFileListProps {
  currentFileBrowserPath: string;
  topRepoPath: string;
  stagedFiles: any;
  unstagedFiles: any;
  untrackedFiles: any;
  app: JupyterLab;
  refresh: any;
  sideBarExpanded: boolean;
  display: boolean;
  currentTheme: string;
}

export class FileList extends React.Component<IFileListProps, IFileListState> {
  constructor(props: IFileListProps) {
    super(props);

    const { commands } = this.props.app;

    this.state = {
      commitMessage: '',
      disableCommit: true,
      showStaged: true,
      showUnstaged: true,
      showUntracked: true,
      contextMenuStaged: new Menu({ commands }),
      contextMenuUnstaged: new Menu({ commands }),
      contextMenuUntracked: new Menu({ commands }),
      contextMenuTypeX: '',
      contextMenuTypeY: '',
      contextMenuFile: '',
      contextMenuIndex: -1,
      contextMenuStage: '',
      selectedFile: -1,
      selectedStage: '',
      selectedDiscardFile: -1,
      disableStaged: false,
      disableUnstaged: false,
      disableUntracked: false,
      disableFiles: false
    };

    /** Add right-click menu options for files in repo 
      * 
      */

    if (!commands.hasCommand(CommandIDs.gitFileOpen)) {
      commands.addCommand(CommandIDs.gitFileOpen, {
        label: 'Open',
        caption: 'Open selected file',
        execute: () => {
          try {
            this.openListedFile(
              this.state.contextMenuTypeX,
              this.state.contextMenuTypeY,
              this.state.contextMenuFile,
              this.props.app
            );
          } catch (err) {}
        }
      });
    }
    if (!commands.hasCommand(CommandIDs.gitFileStage)) {
      commands.addCommand(CommandIDs.gitFileStage, {
        label: 'Stage',
        caption: 'Stage the changes of selected file',
        execute: () => {
          try {
            this.addUnstagedFile(
              this.state.contextMenuFile,
              this.props.topRepoPath,
              this.props.refresh
            );
          } catch (err) {}
        }
      });
    }
    if (!commands.hasCommand(CommandIDs.gitFileTrack)) {
      commands.addCommand(CommandIDs.gitFileTrack, {
        label: 'Track',
        caption: 'Start tracking selected file',
        execute: () => {
          try {
            this.addUntrackedFile(
              this.state.contextMenuFile,
              this.props.topRepoPath,
              this.props.refresh
            );
          } catch (err) {}
        }
      });
    }
    if (!commands.hasCommand(CommandIDs.gitFileUnstage)) {
      commands.addCommand(CommandIDs.gitFileUnstage, {
        label: 'Unstage',
        caption: 'Unstage the changes of selected file',
        execute: () => {
          try {
            if (this.state.contextMenuTypeX !== 'D') {
              this.resetStagedFile(
                this.state.contextMenuFile,
                this.props.topRepoPath,
                this.props.refresh
              );
            }
          } catch (err) {}
        }
      });
    }
    if (!commands.hasCommand(CommandIDs.gitFileDiscard)) {
      commands.addCommand(CommandIDs.gitFileDiscard, {
        label: 'Discard',
        caption: 'Discard recent changes of selected file',
        execute: () => {
          try {
            this.updateSelectedFile(
              this.state.contextMenuIndex,
              this.state.contextMenuStage
            );
            this.updateSelectedDiscardFile(this.state.contextMenuIndex);
            this.toggleDisableFiles();
          } catch (err) {}
        }
      });
    }
    this.state.contextMenuStaged.addItem({ command: CommandIDs.gitFileOpen });
    this.state.contextMenuStaged.addItem({
      command: CommandIDs.gitFileUnstage
    });

    this.state.contextMenuUnstaged.addItem({ command: CommandIDs.gitFileOpen });
    this.state.contextMenuUnstaged.addItem({
      command: CommandIDs.gitFileStage
    });
    this.state.contextMenuUnstaged.addItem({
      command: CommandIDs.gitFileDiscard
    });

    this.state.contextMenuUntracked.addItem({
      command: CommandIDs.gitFileOpen
    });
    this.state.contextMenuUntracked.addItem({
      command: CommandIDs.gitFileTrack
    });
  }

  /** Handle clicks on a staged file
   * 
   */
  handleClickStaged(event: any) {
    event.preventDefault();
    if (event.buttons === 2) {
      <select>
        <option className="jp-Git-switch-branch" value="" disabled>
          Open
        </option>
        <option className="jp-Git-create-branch-line" disabled>
          unstaged this file
        </option>
        <option className="jp-Git-create-branch" value="">
          CREATE NEW
        </option>
      </select>;
    }
  }

  /** Handle right-click on a staged file */
  contextMenuStaged = (
    event: any,
    typeX: string,
    typeY: string,
    file: string,
    index: number,
    stage: string
  ) => {
    event.persist();
    event.preventDefault();
    this.setState(
      {
        contextMenuTypeX: typeX,
        contextMenuTypeY: typeY,
        contextMenuFile: file,
        contextMenuIndex: index,
        contextMenuStage: stage
      },
      () => this.state.contextMenuStaged.open(event.clientX, event.clientY)
    );
  };

  /** Handle right-click on an unstaged file */
  contextMenuUnstaged = (
    event: any,
    typeX: string,
    typeY: string,
    file: string,
    index: number,
    stage: string
  ) => {
    event.persist();
    event.preventDefault();
    this.setState(
      {
        contextMenuTypeX: typeX,
        contextMenuTypeY: typeY,
        contextMenuFile: file,
        contextMenuIndex: index,
        contextMenuStage: stage
      },
      () => this.state.contextMenuUnstaged.open(event.clientX, event.clientY)
    );
  };

  /** Handle right-click on an untracked file */
  contextMenuUntracked = (
    event: any,
    typeX: string,
    typeY: string,
    file: string,
    index: number,
    stage: string
  ) => {
    event.persist();
    event.preventDefault();
    this.setState(
      {
        contextMenuTypeX: typeX,
        contextMenuTypeY: typeY,
        contextMenuFile: file,
        contextMenuIndex: index,
        contextMenuStage: stage
      },
      () => this.state.contextMenuUntracked.open(event.clientX, event.clientY)
    );
  };

  /** Toggle display of staged files */
  displayStaged = (): void => {
    this.setState({ showStaged: !this.state.showStaged });
  };

  /** Toggle display of unstaged files */
  displayUnstaged = (): void => {
    this.setState({ showUnstaged: !this.state.showUnstaged });
  };

  /** Toggle display of untracked files */
  displayUntracked = (): void => {
    this.setState({ showUntracked: !this.state.showUntracked });
  };

  updateSelectedStage = (stage: string): void => {
    this.setState({ selectedStage: stage });
  };

  /** Open a file in the git listing */
  async openListedFile(
    typeX: string,
    typeY: string,
    path: string,
    app: JupyterLab
  ) {
    if (typeX === 'D' || typeY === 'D') {
      showDialog({
        title: 'Open File Failed',
        body: 'This file has been deleted!',
        buttons: [Dialog.warnButton({ label: 'OK' })]
      }).then(result => {
        if (result.button.accept) {
          return;
        }
      });
      return;
    }
    try {
      const leftSidebarItems = app.shell.widgets('left');
      let fileBrowser = leftSidebarItems.next();
      while (fileBrowser.id !== 'filebrowser') {
        fileBrowser = leftSidebarItems.next();
      }
      let gitApi = new Git();
      let prefixData = await gitApi.showPrefix((fileBrowser as any).model.path);
      let underRepoPath = (prefixData as GitShowPrefixResult).under_repo_path;
      let fileBrowserPath = (fileBrowser as any).model.path + '/';
      let openFilePath = fileBrowserPath.substring(
        0,
        fileBrowserPath.length - underRepoPath.length
      );
      if (path[path.length - 1] !== '/') {
        (fileBrowser as any)._listing._manager.openOrReveal(
          openFilePath + path
        );
      } else {
        console.log('Cannot open a folder here');
      }
    } catch (err) {}
  }

  /** Reset all staged files */
  resetAllStagedFiles(path: string, refresh: Function) {
    let gitApi = new Git();
    gitApi.reset(true, null, path).then(response => {
      refresh();
    });
  }

  /** Reset a specific staged file */
  resetStagedFile(file: string, path: string, refresh: Function) {
    let gitApi = new Git();
    gitApi.reset(false, file, path).then(response => {
      refresh();
    });
  }

  /** Add all unstaged files */
  addAllUnstagedFiles(path: string, refresh: Function) {
    let gitApi = new Git();
    gitApi.add(true, null, path).then(response => {
      refresh();
    });
  }

  /** Discard changes in all unstaged files */
  discardAllUnstagedFiles(path: string, refresh: Function) {
    let gitApi = new Git();
    gitApi.checkout(false, false, null, true, null, path).then(response => {
      refresh();
    });
  }

  /** Add a specific unstaged file */
  addUnstagedFile(file: string, path: string, refresh: Function) {
    let gitApi = new Git();
    gitApi.add(false, file, path).then(response => {
      refresh();
    });
  }

  /** Discard changes in a specific unstaged file */
  discardUnstagedFile(file: string, path: string, refresh: Function) {
    let gitApi = new Git();
    gitApi.checkout(false, false, null, false, file, path).then(response => {
      refresh();
    });
  }

  /** Add all untracked files */
  addAllUntrackedFiles(path: string, refresh: Function) {
    let gitApi = new Git();
    gitApi.addAllUntracked(path).then(response => {
      refresh();
    });
  }

  /** Add a specific untracked file */
  addUntrackedFile(file: string, path: string, refresh: Function) {
    let gitApi = new Git();
    gitApi.add(false, file, path).then(response => {
      refresh();
    });
  }

  /** Get the filename from a path */
  extractFilename(path: string): string {
    if (path[path.length - 1] === '/') {
      return path;
    } else {
      let temp = path.split('/');
      return temp[temp.length - 1];
    }
  }

  disableStagesForDiscardAll = () => {
    this.setState({
      disableStaged: !this.state.disableStaged,
      disableUntracked: !this.state.disableUntracked
    });
  };

  updateSelectedDiscardFile = (index: number): void => {
    this.setState({ selectedDiscardFile: index });
  };

  toggleDisableFiles = (): void => {
    this.setState({ disableFiles: !this.state.disableFiles });
  };

  updateSelectedFile = (file: number, stage: string) => {
    this.setState({ selectedFile: file }, () =>
      this.updateSelectedStage(stage)
    );
  };

  render() {
    return (
      <div onContextMenu={event => event.preventDefault()}>
        {this.props.display && (
          <div>
            <GitStage
              heading={'Staged'}
              topRepoPath={this.props.topRepoPath}
              files={this.props.stagedFiles}
              app={this.props.app}
              refresh={this.props.refresh}
              showFiles={this.state.showStaged}
              displayFiles={this.displayStaged}
              moveAllFiles={this.resetAllStagedFiles}
              discardAllFiles={null}
              discardFile={null}
              moveFile={this.resetStagedFile}
              moveFileIconClass={moveFileDownButtonStyle}
              moveFileIconSelectedClass={moveFileDownButtonSelectedStyle}
              moveAllFilesTitle={'Unstage all changes'}
              moveFileTitle={'Unstage this change'}
              openFile={this.openListedFile}
              extractFilename={this.extractFilename}
              contextMenu={this.contextMenuStaged}
              parseFileExtension={parseFileExtension}
              parseSelectedFileExtension={parseSelectedFileExtension}
              selectedFile={this.state.selectedFile}
              updateSelectedFile={this.updateSelectedFile}
              selectedStage={this.state.selectedStage}
              updateSelectedStage={this.updateSelectedStage}
              selectedDiscardFile={this.state.selectedDiscardFile}
              updateSelectedDiscardFile={this.updateSelectedDiscardFile}
              disableFiles={this.state.disableFiles}
              toggleDisableFiles={this.toggleDisableFiles}
              disableOthers={null}
              isDisabled={this.state.disableStaged}
              sideBarExpanded={this.props.sideBarExpanded}
              currentTheme={this.props.currentTheme}
            />
            <GitStage
              heading={'Changed'}
              topRepoPath={this.props.topRepoPath}
              files={this.props.unstagedFiles}
              app={this.props.app}
              refresh={this.props.refresh}
              showFiles={this.state.showUnstaged}
              displayFiles={this.displayUnstaged}
              moveAllFiles={this.addAllUnstagedFiles}
              discardAllFiles={this.discardAllUnstagedFiles}
              discardFile={this.discardUnstagedFile}
              moveFile={this.addUnstagedFile}
              moveFileIconClass={moveFileUpButtonStyle}
              moveFileIconSelectedClass={moveFileUpButtonSelectedStyle}
              moveAllFilesTitle={'Stage all changes'}
              moveFileTitle={'Stage this change'}
              openFile={this.openListedFile}
              extractFilename={this.extractFilename}
              contextMenu={this.contextMenuUnstaged}
              parseFileExtension={parseFileExtension}
              parseSelectedFileExtension={parseSelectedFileExtension}
              selectedFile={this.state.selectedFile}
              updateSelectedFile={this.updateSelectedFile}
              selectedStage={this.state.selectedStage}
              updateSelectedStage={this.updateSelectedStage}
              selectedDiscardFile={this.state.selectedDiscardFile}
              updateSelectedDiscardFile={this.updateSelectedDiscardFile}
              disableFiles={this.state.disableFiles}
              toggleDisableFiles={this.toggleDisableFiles}
              disableOthers={this.disableStagesForDiscardAll}
              isDisabled={this.state.disableUnstaged}
              sideBarExpanded={this.props.sideBarExpanded}
              currentTheme={this.props.currentTheme}
            />
            <GitStage
              heading={'Untracked'}
              topRepoPath={this.props.topRepoPath}
              files={this.props.untrackedFiles}
              app={this.props.app}
              refresh={this.props.refresh}
              showFiles={this.state.showUntracked}
              displayFiles={this.displayUntracked}
              moveAllFiles={this.addAllUntrackedFiles}
              discardAllFiles={null}
              discardFile={null}
              moveFile={this.addUntrackedFile}
              moveFileIconClass={moveFileUpButtonStyle}
              moveFileIconSelectedClass={moveFileUpButtonSelectedStyle}
              moveAllFilesTitle={'Track all untracked files'}
              moveFileTitle={'Track this file'}
              openFile={this.openListedFile}
              extractFilename={this.extractFilename}
              contextMenu={this.contextMenuUntracked}
              parseFileExtension={parseFileExtension}
              parseSelectedFileExtension={parseSelectedFileExtension}
              selectedFile={this.state.selectedFile}
              updateSelectedFile={this.updateSelectedFile}
              selectedStage={this.state.selectedStage}
              updateSelectedStage={this.updateSelectedStage}
              selectedDiscardFile={this.state.selectedDiscardFile}
              updateSelectedDiscardFile={this.updateSelectedDiscardFile}
              disableFiles={this.state.disableFiles}
              toggleDisableFiles={this.toggleDisableFiles}
              disableOthers={null}
              isDisabled={this.state.disableUntracked}
              sideBarExpanded={this.props.sideBarExpanded}
              currentTheme={this.props.currentTheme}
            />
          </div>
        )}
      </div>
    );
  }
}

/** Get the extension of a given file */
export function parseFileExtension(path: string): string {
  if (path[path.length - 1] === '/') {
    return folderFileIconStyle;
  }
  var fileExtension = PathExt.extname(path).toLocaleLowerCase();
  switch (fileExtension) {
    case '.md':
      return markdownFileIconStyle;
    case '.py':
      return pythonFileIconStyle;
    case '.json':
      return jsonFileIconStyle;
    case '.csv':
      return spreadsheetFileIconStyle;
    case '.xls':
      return spreadsheetFileIconStyle;
    case '.r':
      return kernelFileIconStyle;
    case '.yml':
      return yamlFileIconStyle;
    case '.yaml':
      return yamlFileIconStyle;
    case '.svg':
      return imageFileIconStyle;
    case '.tiff':
      return imageFileIconStyle;
    case '.jpeg':
      return imageFileIconStyle;
    case '.jpg':
      return imageFileIconStyle;
    case '.gif':
      return imageFileIconStyle;
    case '.png':
      return imageFileIconStyle;
    case '.raw':
      return imageFileIconStyle;
    default:
      return genericFileIconStyle;
  }
}

/** Get the extension of a given selected file */
export function parseSelectedFileExtension(path: string): string {
  if (path[path.length - 1] === '/') {
    return folderFileIconSelectedStyle;
  }
  var fileExtension = PathExt.extname(path).toLocaleLowerCase();
  switch (fileExtension) {
    case '.md':
      return markdownFileIconSelectedStyle;
    case '.py':
      return pythonFileIconSelectedStyle;
    case '.json':
      return jsonFileIconSelectedStyle;
    case '.csv':
      return spreadsheetFileIconSelectedStyle;
    case '.xls':
      return spreadsheetFileIconSelectedStyle;
    case '.r':
      return kernelFileIconSelectedStyle;
    case '.yml':
      return yamlFileIconSelectedStyle;
    case '.yaml':
      return yamlFileIconSelectedStyle;
    case '.svg':
      return imageFileIconSelectedStyle;
    case '.tiff':
      return imageFileIconSelectedStyle;
    case '.jpeg':
      return imageFileIconSelectedStyle;
    case '.jpg':
      return imageFileIconSelectedStyle;
    case '.gif':
      return imageFileIconSelectedStyle;
    case '.png':
      return imageFileIconSelectedStyle;
    case '.raw':
      return imageFileIconSelectedStyle;
    default:
      return genericFileIconSelectedStyle;
  }
}
```

```typescript
/** jupytelab-git/src/components/FileItem.tsx */
import { JupyterLab } from '@jupyterlab/application';

import {
  changeStageButtonStyle,
  changeStageButtonLeftStyle,
  discardFileButtonStyle
} from '../componentsStyle/GitStageStyle';

import {
  fileStyle,
  selectedFileStyle,
  expandedFileStyle,
  disabledFileStyle,
  fileIconStyle,
  fileLabelStyle,
  fileChangedLabelStyle,
  selectedFileChangedLabelStyle,
  fileChangedLabelBrandStyle,
  fileChangedLabelInfoStyle,
  discardWarningStyle,
  fileButtonStyle,
  fileGitButtonStyle,
  discardFileButtonSelectedStyle,
  cancelDiscardButtonStyle,
  acceptDiscardButtonStyle,
  discardButtonStyle,
  sideBarExpandedFileLabelStyle
} from '../componentsStyle/FileItemStyle';

import { classes } from 'typestyle';

import * as React from 'react';

export interface IFileItemProps {
  topRepoPath: string;
  file: any;
  stage: string;
  app: JupyterLab;
  refresh: any;
  moveFile: Function;
  discardFile: Function;
  moveFileIconClass: Function;
  moveFileIconSelectedClass: string;
  moveFileTitle: string;
  openFile: Function;
  extractFilename: Function;
  contextMenu: Function;
  parseFileExtension: Function;
  parseSelectedFileExtension: Function;
  selectedFile: number;
  updateSelectedFile: Function;
  fileIndex: number;
  selectedStage: string;
  selectedDiscardFile: number;
  updateSelectedDiscardFile: Function;
  disableFile: boolean;
  toggleDisableFiles: Function;
  sideBarExpanded: boolean;
  currentTheme: string;
}

export class FileItem extends React.Component<IFileItemProps, {}> {
  constructor(props: IFileItemProps) {
    super(props);
  }

  checkSelected(): boolean {
    return (
      this.props.selectedFile === this.props.fileIndex &&
      this.props.selectedStage === this.props.stage
    );
  }

  getFileChangedLabel(change: string): string {
    if (change === 'M') {
      return 'Mod';
    } else if (change === 'A') {
      return 'Add';
    } else if (change === 'D') {
      return 'Rmv';
    } else if (change === 'R') {
      return 'Rnm';
    }
  }

  showDiscardWarning(): boolean {
    return (
      this.props.selectedDiscardFile === this.props.fileIndex &&
      this.props.stage === 'Changed'
    );
  }

  getFileChangedLabelClass(change: string) {
    if (change === 'M') {
      if (this.showDiscardWarning()) {
        return classes(fileChangedLabelStyle, fileChangedLabelBrandStyle);
      } else {
        return this.checkSelected()
          ? classes(
              fileChangedLabelStyle,
              fileChangedLabelBrandStyle,
              selectedFileChangedLabelStyle
            )
          : classes(fileChangedLabelStyle, fileChangedLabelBrandStyle);
      }
    } else {
      if (this.showDiscardWarning()) {
        return classes(fileChangedLabelStyle, fileChangedLabelInfoStyle);
      } else {
        return this.checkSelected()
          ? classes(
              fileChangedLabelStyle,
              fileChangedLabelInfoStyle,
              selectedFileChangedLabelStyle
            )
          : classes(fileChangedLabelStyle, fileChangedLabelInfoStyle);
      }
    }
  }

  getFileLableIconClass() {
    if (this.showDiscardWarning()) {
      return classes(
        fileIconStyle,
        this.props.parseFileExtension(this.props.file.to)
      );
    } else {
      return this.checkSelected()
        ? classes(
            fileIconStyle,
            this.props.parseSelectedFileExtension(this.props.file.to)
          )
        : classes(
            fileIconStyle,
            this.props.parseFileExtension(this.props.file.to)
          );
    }
  }

  getFileClass() {
    if (!this.checkSelected() && this.props.disableFile) {
      return classes(fileStyle, disabledFileStyle);
    } else if (this.showDiscardWarning()) {
      classes(fileStyle, expandedFileStyle);
    } else {
      return this.checkSelected()
        ? classes(fileStyle, selectedFileStyle)
        : classes(fileStyle);
    }
  }

  getFileLabelClass() {
    return this.props.sideBarExpanded
      ? classes(fileLabelStyle, sideBarExpandedFileLabelStyle)
      : fileLabelStyle;
  }

  getMoveFileIconClass() {
    if (this.showDiscardWarning()) {
      return classes(
        fileButtonStyle,
        changeStageButtonStyle,
        changeStageButtonLeftStyle,
        fileGitButtonStyle,
        this.props.moveFileIconClass(this.props.currentTheme)
      );
    } else {
      return this.checkSelected()
        ? classes(
            fileButtonStyle,
            changeStageButtonStyle,
            changeStageButtonLeftStyle,
            fileGitButtonStyle,
            this.props.moveFileIconSelectedClass
          )
        : classes(
            fileButtonStyle,
            changeStageButtonStyle,
            changeStageButtonLeftStyle,
            fileGitButtonStyle,
            this.props.moveFileIconClass(this.props.currentTheme)
          );
    }
  }

  getDiscardFileIconClass() {
    if (this.showDiscardWarning()) {
      return classes(
        fileButtonStyle,
        changeStageButtonStyle,
        fileGitButtonStyle,
        discardFileButtonStyle(this.props.currentTheme)
      );
    } else {
      return this.checkSelected()
        ? classes(
            fileButtonStyle,
            changeStageButtonStyle,
            fileGitButtonStyle,
            discardFileButtonSelectedStyle
          )
        : classes(
            fileButtonStyle,
            changeStageButtonStyle,
            fileGitButtonStyle,
            discardFileButtonStyle(this.props.currentTheme)
          );
    }
  }

  getDiscardWarningClass() {
    return discardWarningStyle;
  }

  render() {
    return (
      <div
        className={this.getFileClass()}
        onClick={() =>
          this.props.updateSelectedFile(this.props.fileIndex, this.props.stage)}>
        <button
          className={jp-Git-button ${this.getMoveFileIconClass()}
          title={this.props.moveFileTitle}
          onClick={() => {
            this.props.moveFile(
              this.props.file.to,
              this.props.topRepoPath,
              this.props.refresh
            );
          }}
        />
        <span className={this.getFileLableIconClass()} />
        <span
          className={this.getFileLabelClass()}
          onContextMenu={e => {
            this.props.contextMenu(
              e,
              this.props.file.x,
              this.props.file.y,
              this.props.file.to,
              this.props.fileIndex,
              this.props.stage
            );
          }}
          onDoubleClick={() =>
            this.props.openFile(
              this.props.file.x,
              this.props.file.y,
              this.props.file.to,
              this.props.app
            )}
        >
          {this.props.extractFilename(this.props.file.to)}
          <span className={this.getFileChangedLabelClass(this.props.file.y)}>
            {this.getFileChangedLabel(this.props.file.y)}
          </span>
          {this.props.stage === 'Changed' && (
            <button
              className={jp-Git-button ${this.getDiscardFileIconClass()}
              title={'Discard this change'}
              onClick={() => {
                this.props.toggleDisableFiles();
                this.props.updateSelectedDiscardFile(this.props.fileIndex);
              }}
            />
          )}
        </span>
        {this.showDiscardWarning() && (
          <div className={this.getDiscardWarningClass()}>
            These changes will be gone forever
            <div>
              <button
                className={classes(
                  discardButtonStyle,
                  cancelDiscardButtonStyle
                )}
                onClick={() => {
                  this.props.toggleDisableFiles();
                  this.props.updateSelectedDiscardFile(-1);
                }}
              >
                Cancel
              </button>
              <button
                className={classes(
                  discardButtonStyle,
                  acceptDiscardButtonStyle
                )}
                onClick={() => {
                  this.props.discardFile(
                    this.props.file.to,
                    this.props.topRepoPath,
                    this.props.refresh
                  ),
                    this.props.toggleDisableFiles(),
                    this.props.updateSelectedDiscardFile(-1);
                }}
              >
                Discard
              </button>
            </div>
          </div>
        )}
      </div>
    );
  }
}
```

```typescript
/** jupyterlab-git/src/components/CommitBox.tsx */
import * as React from 'react';

import {
  textInputStyle,
  stagedCommitStyle,
  stagedCommitMessageStyle
} from '../componentsStyle/BranchHeaderStyle';

import { classes } from 'typestyle';

export interface ICommitBoxProps {
  checkReadyForSubmit: Function;
  stagedFiles: any;
  commitAllStagedFiles: Function;
  topRepoPath: string;
  refresh: Function;
}

export interface ICommitBoxState {
  value: string;
  disableSubmit: boolean;
}

export class CommitBox extends React.Component<
  ICommitBoxProps,
  ICommitBoxState
> {
  constructor(props: ICommitBoxProps) {
    super(props);
    this.state = {
      value: '',
      disableSubmit: true
    };
  }

  /** Prevent enter key triggered 'submit' action during commit message input */
  onKeyPress(event: any): void {
    if (event.which === 13) {
      event.preventDefault();
      this.setState({ value: this.state.value + '\n' });
    }
  }

  /** Initalize commit message input box */
  initializeInput = (): void => {
    this.setState({
      value: '',
      disableSubmit: true
    });
  };

  /** Handle input inside commit message box */
  handleChange = (event: any): void => {
    if (event.target.value && event.target.value !== '') {
      this.setState({
        value: event.target.value,
        disableSubmit: false
      });
    } else {
      this.setState({
        value: event.target.value,
        disableSubmit: true
      });
    }
  };

  render() {
    return (
      <form
        className={stagedCommitStyle}
        onKeyPress={event => this.onKeyPress(event)}
      >
        <textarea
          className={classes(textInputStyle, stagedCommitMessageStyle)}
          disabled={this.props.stagedFiles.length === 0}
          placeholder={
            this.props.stagedFiles.length === 0
              ? 'Stage your changes before commit'
              : 'Input message to commit staged changes'
          }
          value={this.state.value}
          onChange={this.handleChange}
        />
        <input
          className={this.props.checkReadyForSubmit(
            this.state.disableSubmit,
            this.props.stagedFiles.length
          )}
          type="button"
          title="Commit"
          disabled={this.state.disableSubmit}
          onClick={() => {
            this.props.commitAllStagedFiles(
              this.state.value,
              this.props.topRepoPath,
              this.props.refresh
            ),
              this.initializeInput();
          }}
        />
      </form>
    );
  }
}
```

```typescript
/** jupyterlab-git/src/components/BranchHeader.tsx */
import * as React from 'react';

import { Git } from '../git';

import { CommitBox } from './CommitBox';

import { NewBranchBox } from './NewBranchBox';

import {
  branchStyle,
  branchLabelStyle,
  branchDropdownButtonStyle,
  newBranchButtonStyle,
  headerButtonDisabledStyle,
  branchListItemStyle,
  stagedCommitButtonStyle,
  stagedCommitButtonReadyStyle,
  stagedCommitButtonDisabledStyle,
  smallBranchStyle,
  expandedBranchStyle,
  openHistorySideBarButtonStyle,
  openHistorySideBarIconStyle,
  branchHeaderCenterContent
} from '../componentsStyle/BranchHeaderStyle';

import { classes } from 'typestyle';

export interface IBranchHeaderState {
  dropdownOpen: boolean;
  showCommitBox: boolean;
  showNewBranchBox: boolean;
}

export interface IBranchHeaderProps {
  currentFileBrowserPath: string;
  topRepoPath: string;
  currentBranch: string;
  stagedFiles: any;
  data: any;
  refresh: any;
  disabled: boolean;
  toggleSidebar: Function;
  showList: boolean;
  currentTheme: string;
}

export class BranchHeader extends React.Component<
  IBranchHeaderProps,
  IBranchHeaderState
> {
  interval: any;
  constructor(props: IBranchHeaderProps) {
    super(props);
    this.state = {
      dropdownOpen: false,
      showCommitBox: true,
      showNewBranchBox: false
    };
  }

  /** Commit all staged files */
  commitAllStagedFiles = (message: string, path: string): void => {
    if (message && message !== '') {
      let gitApi = new Git();
      gitApi.commit(message, path).then(response => {
        this.props.refresh();
      });
    }
  };

  /** Update state of commit message input box */
  updateCommitBoxState(disable: boolean, numberOfFiles: number) {
    if (disable) {
      if (numberOfFiles === 0) {
        return classes(
          stagedCommitButtonStyle,
          stagedCommitButtonDisabledStyle
        );
      } else {
        return classes(stagedCommitButtonStyle, stagedCommitButtonReadyStyle);
      }
    } else {
      return stagedCommitButtonStyle;
    }
  }

  /** Switch current working branch */
  async switchBranch(branchName: string) {
    let gitApi = new Git();
    await gitApi.checkout(
      true,
      false,
      branchName,
      false,
      null,
      this.props.currentFileBrowserPath
    );
    this.toggleSelect();
    this.props.refresh();
  }

  createNewBranch = async (branchName: string) => {
    let gitApi = new Git();
    await gitApi.checkout(
      true,
      true,
      branchName,
      false,
      null,
      this.props.currentFileBrowserPath
    );
    this.toggleNewBranchBox();
    this.props.refresh();
  };

  toggleSelect() {
    this.props.refresh();
    if (!this.props.disabled) {
      this.setState({
        showCommitBox: !this.state.showCommitBox,
        dropdownOpen: !this.state.dropdownOpen
      });
    }
  }

  getBranchStyle() {
    if (this.state.dropdownOpen) {
      return classes(branchStyle, expandedBranchStyle);
    } else {
      return this.props.showList
        ? branchStyle
        : classes(branchStyle, smallBranchStyle);
    }
  }

  toggleNewBranchBox = (): void => {
    this.props.refresh();
    if (!this.props.disabled) {
      this.setState({
        showNewBranchBox: !this.state.showNewBranchBox,
        dropdownOpen: false
      });
    }
  };

  render() {
    return (
      <div className={this.getBranchStyle()}>
        <button
          className={openHistorySideBarButtonStyle}
          onClick={() => this.props.toggleSidebar()}
          title={'Show commit history'}
        >
          History
          <span className={openHistorySideBarIconStyle} />
        </button>
        <div className={branchHeaderCenterContent}>
          <h3 className={branchLabelStyle}>{this.props.currentBranch}</h3>
          <div
            className={
              this.props.disabled
                ? classes(
                    branchDropdownButtonStyle(this.props.currentTheme),
                    headerButtonDisabledStyle
                  )
                : branchDropdownButtonStyle(this.props.currentTheme)
            }
            title={'Change the current branch'}
            onClick={() => this.toggleSelect()}
          />
          {!this.state.showNewBranchBox && (
            <div
              className={
                this.props.disabled
                  ? classes(
                      newBranchButtonStyle(this.props.currentTheme),
                      headerButtonDisabledStyle
                    )
                  : newBranchButtonStyle(this.props.currentTheme)
              }
              title={'Create a new branch'}
              onClick={() => this.toggleNewBranchBox()}
            />
          )}
          {this.state.showNewBranchBox &&
            this.props.showList && (
              <NewBranchBox
                createNewBranch={this.createNewBranch}
                toggleNewBranchBox={this.toggleNewBranchBox}
              />
            )}
        </div>
        {this.state.dropdownOpen && (
          <div>
            {this.props.data.map((branch: any, branchIndex: number) => {
              return (
                <li
                  className={branchListItemStyle}
                  key={branchIndex}
                  onClick={() => this.switchBranch(branch.name)}
                >
                  {branch.name}
                </li>
              );
            })}
          </div>
        )}
        {this.state.showNewBranchBox && (
          <div>Branching from {this.props.currentBranch}</div>
        )}
        {this.state.showCommitBox &&
          this.props.showList && (
            <CommitBox
              checkReadyForSubmit={this.updateCommitBoxState}
              stagedFiles={this.props.stagedFiles}
              commitAllStagedFiles={this.commitAllStagedFiles}
              topRepoPath={this.props.topRepoPath}
              refresh={this.props.refresh}
            />
          )}
      </div>
    );
  }
}

```

