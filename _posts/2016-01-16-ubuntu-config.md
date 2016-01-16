---
layout: default
author: xfoolish
create: 2016-01-16
title: 程序员的自我修养——Ubuntu环境配置
excerpt: <p>工欲善其事，必先利其器。一个舒适的开发环境不仅可以显得专业有逼格，还能大大提高我们码代码的热情。</p> <p>因此，对开发环境的配置是一个装逼的程序员的自我修养。</p>
---

<div class="blog-wrapper wrapper">
	<div class="blog">
		<div class="blog-content">
			<h2 class="blog-header"><a href="#">{{ page.title }}</a></h2>

			<div class="blog-meta">
				<span class="date-span"><i class="fa fa-lg fa-clock-o"></i><a href="#">{{ page.create }}</a></span>
			</div>

			<div class="clearfix"></div>

			<div class="main-content">
				<p>本文以配置Ubuntu 14.04为例，包括主题环境的配置、开发工具的选择两部分内容。</p>

				<h1>主题环境配置</h1>
				<p>首先，Ubuntu安装后的样子是这样的：</p>
				<img src="/img/b1/1.png"/>
				<p>经过配置后的样子是这样的：</p>
				<img src="/img/b1/2.png"/>
				<p>下面将详细介绍如何进行配置。</p>

				<h2>主题选择</h2>

				<p>这里选择<a href="https://numixproject.org/">Numix Project</a>中的主题，其中包括付费和免费的主题，我们只使用其中免费的主题。</p>

				<p>
				首先安装其中的Numix-Circle Linux Desktop Icon Theme：

				<div class="commands">
				<span class="command"><span class="op">&gt;</span>sudo apt-add-repository ppa:numix/ppa</span>
				<span class="command"><span class="op">&gt;</span>sudo apt-get update</span>
				<span class="command"><span class="op">&gt;</span>sudo apt-get install numix-icon-theme-circle</span>
				</div>
				</p>

				<p>
				再下载一个桌面壁纸：
				<div class="commands">
				<span class="command"><span class="op">&gt;</span>sudo apt-get install numix-wallpaper-aurora</span>
				</div>
				</p>

				<p>
				到目前位置主题和桌面已经安装完成，现在使用unity-tweak-tool进行配置，Ubuntu默认没有安装该工具，使用包管理工具进行安装并启动（也可通过Dash进行启动）：
				<div class="commands">
				<span class="command"><span class="op">&gt;</span>sudo apt-get install unity-tweak-tool</span>
				<span class="command"><span class="op">&gt;</span>unity-tweak-tool</span>
				</div>
				</p>

				<p>启动后进入如下管理工具，在其中对Appearance &gt; Theme和Appearance &gt; Icons进行配置，选择刚才安装的Numix主题。再设置顶部Menubar的透明度，通过Unity &gt; Panel进行配置</p>
				<img src="/img/b1/3.png"/>

				<p>最后更换桌面壁纸，进入系统设置工具（在Dash中进入System Settings），选择刚才下载的壁纸。</p>
				<img src="/img/b1/4.png"/>

				<p>现在来给顶部的menubar添加几个插件，即Indicator Applets，这里只安装my-weather-indicator和indicator-multiload，需要其他的indicator可以继续添加。
				<div class="commands">
				<span class="command"><span class="op">&gt;</span>sudo add-apt-repository ppa:atareao/atareao</span>
				<span class="command"><span class="op">&gt;</span>sudo apt-get update</span>
				<span class="command"><span class="op">&gt;</span>sudo apt-get install my-weather-indicator</span>
				<span class="command"><span class="op">&gt;</span>sudo apt-get install indicator-multiload</span>
				</div>

				</p>

				<img src="/img/b1/5.png"/>

				<p>到目前为止，我们的桌面已经看上去很舒服了，至少和Ubuntu原来的相比。</p>

				<h1>开发环境配置</h1>
				<p>完成了主题环境的配置，我们再来选择开发工具并进行相应配置。</p>

				<h2>git，github和bitbucket</h2>
				<p>这是一个开源的时代，这是一个幸福的时代。今天，优秀的开源项目俯拾即是，作为一个程序员，从来没有现在这么简单和高效。无论进行什么开发，我们基本上都可以在开源的世界中找到帮助。</p>

				<p>所以，如果你还没用过git和github，那么你要不是个新手，要不就是一个不合格的程序员。</p>

				<p>对于git，虽然现在已经有各种强大的可视化git工具可以使用，但我还是建议使用命令行进行操作，这样可以加深git的工作原理，熟悉各种命令的使用。毕竟，无论什么时候，命令行都是随手可用的工具。</p>

				<p><a href="https://github.com/">github</a>是一个git的代码托管平台，再这里我们可以找到各种优秀的开源项目，也可以将自己的项目托管其中，但对于普通用户，托管在github上的项目必须是公开的，私人的项目需要付费用户才能进行托管。</p>

				<p>而对于私人项目，并且开发团队不大，可以选择<a href="https://bitbucket.org/">bitbucket</a>进行托管。</p>

				<h2>终端配置</h2>

				<p>你的终端是不是这样的：</p>
				<img src="/img/b1/6.png"/>
				<p>而别人的终端却是这样的：</p>
				<img src="/img/b1/7.png"/>
				<p>现在我们来对终端进行配置：</p>

				<h3>Zsh, Oh-My-Zsh</h3>
				<p>你是否还在用系统默认的bash，但是大牛们用的却是zsh。至于为什么用zsh，可以自行进行google，zsh的有点至少包括：更强的自动不全、优化的模式识别、全面可定制。通过如下命令来安装zsh并修改默认的shell，其中对shell的更改需要注销重新登录后才能生效。</p>

				<div class="commands">
				<span class="command"><span class="op">&gt;</span>sudo apt-get install zsh</span>
				<span class="command"><span class="op">&gt;</span>chsh -s $(which zsh)</span>
				</div>

				<p>Oh-My-Zsh是一个用来管理zsh配置的开源框架，它包含了丰富的主题、功能、插件和一些你意想不到的东西。安装起来也相当简单：
				<div class="commands">
				<span class="command"><span class="op">&gt;</span>sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"</span>
				</div>
				</p>

				<p>安装好zsh和Oh-My-Zsh后，我们可以通过~/.zshrc来对zsh进行配置，例如主题和插件配置。具体的配置可以借助于google或github。</p>
				<img src="/img/b1/8.png"/>

				<h3>tmux</h3>
				<p>还在苦于打开多个终端，在终端中来回切换吗，终端的分屏复用可以帮助你，而<a href="https://tmux.github.io/">tmux</a>就是其中的杰出代表。它的安装和使用也非常简单：</p>

				<p>
				<div class="commands">
				<span class="command"><span class="op">&gt;</span>sudo apt-get install tmux</span>
				<span class="command"><span class="op">&gt;</span>tmux</span>
				</div>
				</p>

				<h2>Vim配置</h2>
				<p>Vim编辑器可以说是Linux世界最出名最优秀的编辑器之一了，但是对于初学者来说，它并不是那么方便使用，不过一旦你熟悉了它，你就会发现它的美并且爱上它。当然为了使Vim使用起来上手，必要的配置是并不可少的，这也正式Vim的强大和美好之处。这里不对具体的配置细节进行介绍，只寻求快速上手。</p>
				<p>最简单的办法就是直接使用别人的配置，可以通过github寻找这样的配置，并选择star教多的，例如我选择的就是star最多的<a href="https://github.com/amix">amix</a>的配置。</p>
				<img src="/img/b1/9.png"/>

				<h2>文本编辑器</h2>
				<p>如果你进行Java，C++，Android等开发，那么可能Eclipse，IntelliJ等IDE更适合你。我这里说的文本编辑器，只是纯粹的文本编辑，并不涉及编译、调试等复杂功能。当然一个好的编辑器必须实现代码的高亮显示和自动缩进、项目文件的导航、分屏、插件等功能。</p>

				<p>对于这样的编辑器，可以使用<a href="http://www.sublimetext.com/">Sublime Text</a>和<a href="https://atom.io/">atom</a>，其中我更推荐使用atom。</p>
				<img src="/img/b1/10.png"/>
			</div>
		</div>
	</div>
</div>
