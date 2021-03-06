# 启用spice\(M版\)

#### 背景

OpenStack默认无法通过spice客户端直连，但实际上运行的qemu实例启用了spice，用以支持spice-html5。  
通过查看运行的qemu实例进程信息，就能看到如下实例的虚拟机使用的spice端口为6001

```text
/usr/libexec/qemu\-kvm -name guest=instance-000007c3 -vnc 0.0.0.0:100 -k en-us -spice port=6001,addr=0.0.0.0,disable-ticketing,seamless-migration=on
```

所以，通过客户端连接虚拟机理论上是可行的。

#### 操作步骤：

**修改计算节点防火墙**

通过防火墙规则允许spice的端口范围，可从5900开始开放一个范围，允许外界访问。  
此时通过spice客户端输入地址 spice://ip:port 就可以连接spice客户端了。

**连接安全**

spice直连没有密码，权限过大。qemu是支持spice设置密码的, 通过虚拟机实例的配置文件可以看到

```text
<graphics type='spice' autoport='yes' listen='0.0.0.0' keymap='en-us'>
      <listen type='address' address='0.0.0.0'/>
 </graphics>
```

但遗憾的是OpenStack创建虚拟机时并没有接口可以设置password选项。要增加password选项，需要修改OpenStack源码。需要修改两个文件。

1. `nova/virt/libvirt/driver.py` 通过在创建虚拟机时设置元数据`spice_passwd`项，并在创建虚拟机时使用该项完成。需要修改`_get_guest_config`方法，关键代码

```text
spice_passwd = instance.metadata.get('spice_passwd')
if (CONF.spice.enabled and
        virt_type not in ('lxc', 'uml', 'xen')):
    graphics = vconfig.LibvirtConfigGuestGraphics()
    graphics.type = "spice"
    graphics.keymap = CONF.spice.keymap
    graphics.listen = CONF.spice.server_listen
    
    if spice_passwd:
        graphics.passwd = spice_passwd
    guest.add_device(graphics)
    add_video_driver = True
```

1. `nova/virt/libvirt/config.py` 需要修改`LibvirtConfigGuestGraphics`，关键代码

```text
class LibvirtConfigGuestGraphics(LibvirtConfigGuestDevice):

    def __init__(self, **kwargs):
        super(LibvirtConfigGuestGraphics, self).__init__(root_name="graphics",
                                                         **kwargs)

        self.type = "vnc"
        self.autoport = True
        self.keymap = None
        self.listen = None
        self.passwd = None

    def format_dom(self):
        dev = super(LibvirtConfigGuestGraphics, self).format_dom()

        dev.set("type", self.type)
        if self.autoport:
            dev.set("autoport", "yes")
        else:
            dev.set("autoport", "no")
        if self.keymap:
            dev.set("keymap", self.keymap)
        if self.listen:
            dev.set("listen", self.listen)
        if self.passwd:
            dev.set("passwd", self.passwd)
        return dev
```

**扩展API，通过API获取连接信息**

1. 新增API，增加`nova/api/openstack/compute/server_connectinfo.py`

```text
import six
from webob import exc

from nova.api.openstack import common
from nova.api.openstack.compute.schemas import server_metadata
from nova.api.openstack import extensions
from nova.api.openstack import wsgi
from nova.api import validation
from nova import compute
from nova import exception
from nova.i18n import _
from nova.compute import rpcapi as compute_rpcapi

ALIAS = 'os-server-connectinfo'
authorize = extensions.os_compute_authorizer(ALIAS)


class ServerConnectinfoController(wsgi.Controller):
    """The server connect info API controller for the OpenStack API."""

    def __init__(self):
        self.compute_api = compute.API(skip_policy_check=True)
        self.compute_rpcapi = compute_rpcapi.ComputeAPI()
        self.handlers = {'vnc': self.compute_rpcapi.get_vnc_console,
                         'spice': self.compute_rpcapi.get_spice_console,
                         'rdp': self.compute_rpcapi.get_rdp_console,
                         'serial': self.compute_rpcapi.get_serial_console,
                         'mks': self.compute_rpcapi.get_mks_console}
        super(ServerConnectinfoController, self).__init__()

    def _get_connect_info(self, context, server_id, body):
        instance = common.get_instance(self.compute_api, context, server_id)
        try:
            protocol = body['os-getServerConnectinfo']['protocol']
            console_type = body['os-getServerConnectinfo']['type']
            handler = self.handlers.get(protocol)
            connect_info = handler(context,
                instance=instance, console_type=console_type)
        except exception.InstanceNotFound:
            msg = _('Server does not exist')
            raise exc.HTTPNotFound(explanation=msg)
        
        return connect_info

    @extensions.expected_errors(404)
    def index(self, req, server_id):
        """Returns the list of connectinfo for a given instance."""
        context = req.environ['nova.context']
        authorize(context, action='index')
        return {'connectinfo': self._get_connect_info(context, server_id)}

    @extensions.expected_errors(404)
    def create(self, req, server_id, body):
        """Return a single connectinfo item."""
        context = req.environ['nova.context']
        authorize(context, action='create')
        data = self._get_connect_info(context, server_id, body)

        try:
            return {'connectinfo': data}
        except KeyError:
            msg = _("Connectinfo item was not found")
            raise exc.HTTPNotFound(explanation=msg)

class ServerConnectinfo(extensions.V21APIExtensionBase):
    """Server Connectinfo API."""
    name = "ServerConnectinfo"
    alias = ALIAS
    version = 1

    def get_resources(self):
        parent = {'member_name': 'server',
                  'collection_name': 'servers'}
        resources = [extensions.ResourceExtension(ALIAS,
                                                  ServerConnectinfoController(),
                                                  parent=parent)]
        return resources

    def get_controller_extensions(self):
        return []
```

1. 在entry\_points.txt中`[nova.api.v21.extensions]`新增

```text
server_connectinfo  = nova.api.openstack.compute.server_connectinfo:ServerConnectinfo
```

