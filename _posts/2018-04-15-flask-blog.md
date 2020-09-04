---
layout: post
title: 对 Flask Web 博客应用的扩展
categories: 物理
tags: Python
---
{:toc}

照着 Flask Web 开发完成的博客项目后，发现有些功能需要进一步完善，包括：

- 自定义头像
- 登录时输入验证码
- 友好的用户编辑器

## 1. 自定义头像

项目中使用了与邮箱绑定的 Gravatar 图形作为用户头像。Gravatar 提供的头像比较简陋，而且可能由于网络问题无法生成头像。多数社交网站和博客提供用户自定义头像功能，因此自己加上了自定义头像的功能。

### 添加头像路径字段

保留原有的avatar_hash字段，定义 `real_avatar` 字段用来存储头像地址。

```python
# models.py
class User(UserMixin, db.Model):
    # ...
    real_avatar = db.Column(db.String(128), default = None) 
```

### 上传文件表单

使用 `FileField` 类创建了上传文件的表单字段。网上也有人把头像修改和编辑个人资料集成到一个表单里，但通常来说用户资料修改和头像修改是独立的功能，而且资料修改有用户和管理员两个级别，集成到一块对程序修改较多。

```python
# main/forms.py
class ChangeAvatarForm(FlaskForm):
    avatar = FileField('')
    submit = SubmitField('Submit')
```

### 修改头像的视图函数

提交上传文件的表单后，可通过request.files字典中获得上传的文件对象，该对象的filename属性为文件名，从文件名中可提取后缀名，判断上传的文件是否是允许的格式。确认格式无误后，将文件保存到upload_folder文件夹中，文件名为“用户名+后缀名”，由于每个用户的用户名不同，因此可以使用便于识别的用户名来命名头像，也可以使用用户id命名头像文件。


```python
# main/views.py
@main.route('/change-avatar', methods=['GET', 'POST'])
@login_required
def change_avatar():
    '''修改头像'''
    form = ChangeAvatarForm()
    if form.validate_on_submit():
        # 文件对象
        avatar = request.files['avatar']
        fname = avatar.filename     
        # 存储路径 
        upload_folder = current_app.config['UPLOAD_FOLDER']      
        # 允许格式
        allowed_extensions = ['png', 'jpg', 'jpeg', 'gif']      
        # 后缀名
        fext = fname.rsplit('.',1)[-1] if '.' in fname else ''
        # 判断是否符合要求
        if fext not in allowed_extensions:   
            flash('File error.')
            return redirect(url_for('.user', username=current_user.username))
        # 路径+用户名+后缀名
        target = '{}{}.{}'.format(upload_folder, current_user.username, fext)
        avatar.save(target)
        current_user.real_avatar = '/static/avatars/{}.{}'.format(current_user.username, fext)
        db.session.add(current_user)
        flash('Your avatar has been updated.')
        return redirect(url_for('.user', username = current_user.username))
    return render_template('change_avatar.html', form=form)
```

配置文件中配置头像文件保存路径：

```python
# config.py
class Config:
    UPLOAD_FOLDER = os.getcwd() + '/app/static/avatars/'
```

### 前端的修改

在change_avatar.html页面渲染ChangeAvatarForm表单

{% raw %}
```html
<!-- templates/change_avatar.html -->
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}

{% block title %}Flasky - Change Avatar{% endblock %}

{% block page_content %}
<div class="page-header">
    <h1>Change Your Avatar</h1>
</div>
<div class="col-md-4">
    {{ wtf.quick_form(form) }}
</div>
{% endblock %}
```
{% endraw %}


在个人用户页面，可直接使用头像图片作为链接，地址是头像上传页面。由于用户资料页面是开放的，只有用户本人或管理员访问时才加上此链接。同时为了和原有的Gravatar头像功能兼容，这里进行了判断：当用户上传了自己的头像时，显示自定义头像，否则显示默认的Gravatar头像。同时，还将自定义头像的尺寸与Gravatar头像的尺寸保持一致，显得更协调些。也可以不用Gravatar头像，像多数网站一样设置一个默认的头像。

除了用户页面外，账户、博客文章、评论这几处也会用到小头像，要予以修改，保持一致。


{% raw %}
```html
<!-- templates/user.html -->
<a {% if user == current_user or current_user.is_administrator() %}href="{{ url_for('.change_avatar') }}"{% endif %}>
{% if user.real_avatar %}
    <img class="img-rounded profile-thumbnail" src="{{ user.real_avatar }}" height="256" width="256" >
{% else %}
    <img class="img-rounded profile-thumbnail" src="{{ user.gravatar(size=256) }}">
{% endif %}
</a>   
```
{% endraw %}

### 测试

![](/assets/img/flask_bolg_avatar.png)

点击大头像即可进入上传头像页面。目前用户提供的头像需要自行截切成正方形，否则头像会压扁或拉长，很难看。一般的社交网站的头像修改界面往往提供图像的预览和截切功能，主要和前端相关，后续也可将该部分功能集成到程序里。

![](/assets/img/flask_change.png){:width="60%"}


## 2. 登录验证码

使用了PIL模块生成验证码图片，并通过Flask的session机制，进行验证码验证。

### 生成验证码

使用string模块：string.ascii_letters+string.digits构造了验证码字符组合。使用的PIL模块，构建了图形对象，并进行划线和高斯模糊处理。字体文件可单独保存到工程里。绘制字符串时，draw.text的前两个参数为字符的位置，可以设置为随机数，使验证码各字符的位置不固定，并且相邻字符略有重合。get_verify_code返回了图形对象和字符串。

```python
import random
import string
from PIL import Image, ImageFont, ImageDraw, ImageFilter


def rndColor():
    '''随机颜色'''
    return (random.randint(32, 127), random.randint(32, 127), random.randint(32, 127))

def gene_text():
    '''生成4位验证码'''
    return ''.join(random.sample(string.ascii_letters+string.digits, 4))

def draw_lines(draw, num, width, height):
    '''划线'''
    for num in range(num):
        x1 = random.randint(0, width / 2)
        y1 = random.randint(0, height / 2)
        x2 = random.randint(0, width)
        y2 = random.randint(height / 2, height)
        draw.line(((x1, y1), (x2, y2)), fill='black', width=1)

def get_verify_code():
    '''生成验证码图形'''
    code = gene_text()
    # 图片大小120×50
    width, height = 120, 50
    # 新图片对象
    im = Image.new('RGB',(width, height),'white')
    # 字体
    font = ImageFont.truetype('app/static/arial.ttf', 40)
    # draw对象
    draw = ImageDraw.Draw(im)
    # 绘制字符串
    for item in range(4):
        draw.text((5+random.randint(-3,3)+23*item, 5+random.randint(-3,3)),
                  text=code[item], fill=rndColor(),font=font )
    # 划线
    draw_lines(draw, 2, width, height)
    # 高斯模糊
    im = im.filter(ImageFilter.GaussianBlur(radius=1.5))
    return im, code
```

### 表单类

为LoginForm增加一个verify_code字段，用来输入验证码。

```python
class LoginForm(FlaskForm):
    email = StringField('Email', validators=[DataRequired(), Length(1, 64),
                                             Email()],)
    password = PasswordField('Password', validators=[DataRequired()])
    verify_code = StringField('VerifyCode', validators=[DataRequired()])
    remember_me = BooleanField('Keep me logged in')
    submit = SubmitField('Log In')
```


### 视图函数 

使用io.BytesIO对象将验证码图片转化为二进制形式，直接作为响应返回前端。设置首部字段的内容格式，这样二进制的内容就能以图形形式在页面中显示。验证码字符串保存在flask.session对象中，对session的操作就像处理字典一样。程序内部使用设置中的SECRET_KEY对session数据加密后，存储在cookie中。

```python
from io import BytesIO
@auth.route('/code')
def get_code():
    image, code = get_verify_code()
    # 图片以二进制形式写入
    buf = BytesIO()
    image.save(buf, 'jpeg')
    buf_str = buf.getvalue()
    # 把buf_str作为response返回前端，并设置首部字段
    response = make_response(buf_str)
    response.headers['Content-Type'] = 'image/gif'
    # 将验证码字符串储存在session中
    session['image'] = code
    return response
```

 在登录的视图函数中，添加验证码验证功能。注意一般验证码是不区分大小写的，这里将输入的验证码和session中保存的验证码字符串都转换成小写后再作判断。

```python
@auth.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        user = User.query.filter_by(email=form.email.data).first()
        if session.get('image').lower() != form.verify_code.data.lower():
            flash('Wrong verify code.')
            return render_template('auth/login.html', form=form)
        if user is not None and user.verify_password(form.password.data):
            login_user(user, form.remember_me.data)
            return redirect(request.args.get('next') or url_for('main.index'))
        flash('Invalid username or password.')
    return render_template('auth/login.html', form=form)
```

### 前端

在前端中加入了验证码图形的路径，该路径指定为生成图形响应的视图函数。当点击验证码图片时，验证码会予以更新。

{% raw %}
```html
{{ wtf.quick_form(form) }}
<img class="verify_code" src="/auth/code " onclick="this.src='/auth/code?'+ Math.random()">
```
{% endraw %}

调整下布置，最终登录表单会显示成这个样子：

<image src="/assets/img/flask_login_form.png" style="width: 500px; margin: 5px"></image>


下面是不同方式生成的验证码：

- 无特效

<image src="/assets/img/code_normal.png" style="width: 100px; margin: 5px"></image>

- 高斯模糊

<image src="/assets/img/code_gauss.png" style="width: 100px; margin: 5px"></image>

- 增加划线

<image src="/assets/img/code_gauss.png" style="width: 100px; margin: 5px"></image>


## 3 Tinymce编辑器

之前Flask博客的文本编辑器使用的是markdown，对不熟悉该语法的普通用户不够友好，因此这里为博客添加个简单易用的Tinymce文本编辑器。

### 项目中添加Tinymce

下载好Tinymce包以及语言包，并添加到项目中。`tinymce_setup.js` 是配置文件，设置了文本编辑器的语言、按钮等。


### 编辑器表单

为了和其它表单的风格保持一致，这里仍使用了Flask-wtf表单。配置文件tinymce_setup.js中标识了id为content的标签作为Tinymce编辑器显示，这里为editor字段添加了相应的指示。测试发现，表单的editor显示为Tinymce后，使用验证函数无法对输入进行判断，这里将输入的判断放入视图函数中。

```python
class EditorForm(FlaskForm):
    title = StringField('标题', validators=[DataRequired(),  Length(1, 64)])
    editor = TextAreaField('正文', id = 'content')
    submit = SubmitField('发表')
```

### 视图函数
使用request.method判断请求为POST后，判断输入是否为空，若无输入则给予flask消息提醒。若已登录的用户具有写博客的权限，则输入的内容作为Post的body_html属性创建新的博客。Tinymce将输入的文本内容转为html代码，因此这里使用body_html，而不使用body。

```python
@main.route('/editor', methods=['GET', 'POST'])
@login_required
def editor():
    ''' 编辑器界面 '''
    form = EditorForm()
    if request.method == 'POST':
        if not form.editor.data:
            flash('Write something.')
            return render_template('editor.html', form=form)
        if current_user.can(Permission.WRITE_ARTICLES):
            print(request.form)
            post = Post(title=request.form['title'],
                        body_html=request.form['editor'],
                        author=current_user._get_current_object())
            db.session.add(post)
            db.session.commit()
            return redirect(url_for('.post', id=post.id))
    return render_template('editor.html', form = form)
```

### 编辑器页面

在editor.html中加入tinymce.min.js、tinymce_setup.js这两个文件。使用wtf.quick_form渲染编辑器表单。

{% raw %}
```python
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}
{% block title %}Editor{% endblock %}
{% block head %}
{{ super() }}
    <script src="{{ url_for('static', filename='tinymce/js/tinymce/tinymce.min.js') }}"></script>
    <script src="{{ url_for('static', filename='js/tinymce_setup.js') }}"></script>
{% endblock %}

{% block page_content %}
{{ wtf.quick_form(form) }}
{% endblock %}
```
{% endraw %}
 

编辑器界面显示如下：

![](/assets/img/flask_code_editor.png){:width="80%"}

### 代码高亮

为了保证提交前后的显示是一样的，仍想使用与tinyMCE编辑窗口中的样式来渲染提交后的页面。需要手动配置js和css文件。下载渲染程序并添加到文章页的html文件中：

{% raw %}
```html
{% block head %}
    {{ super() }}
    <link rel="stylesheet" type="text/css" href="{{ url_for('static', filename='prism.css') }}">
    <script src="{{ url_for('static', filename='js/prism.js') }}"></script>
{% endblock %}
```
{% endraw %}

效果如下：

![](/assets/img/flask_code_highlight.png){:width="80%"}
