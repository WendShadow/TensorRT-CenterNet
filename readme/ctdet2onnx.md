1. Replace THC-based DCNv2 with ATen-based DCNv2. 
If it is not replaced, you will get (TypeError: int() not supported on cdata 'struct THLongTensor *') when converting onnx, and I have no idea to solve it.
So I use DCNv2 from mmdetection.
    * copy the dcn to lib/models/netowrks
        ```bash
        cp -r dcn lib/models/netowrks
        ```
    * upgrade pytorch to 1.0+
    * complie Deform Conv
        ```bash
        cd lib/models/netowrks/dcn
        python setup.py build_ext --inplace
        ``` 

2. Add symbolic to DeformConvFunction.
    ```python
    class ModulatedDeformConvFunction(Function):
    
        @staticmethod
        def symbolic(g, input, offset, mask, weight, bias,stride,padding,dilation,groups,deformable_groups):
            return g.op("DCNv2", input, offset, mask, weight, bias,
                        stride_i = stride,padding_i = padding,dilation_i = dilation,
                        groups_i = groups,deformable_group_i = deformable_groups)
        @staticmethod
        def forward(ctx,
                    input,
                    offset,
                    mask,
                    weight,
                    bias=None,
                    stride=1,
                    padding=0,
                    dilation=1,
                    groups=1,
                    deformable_groups=1):
                    pass#.......
    ```
    Now you can convert the model using Deform Conv to onnx.
    
3. For Centernet.
    * convert [ctdet_coco_dla_2x.pth](https://github.com/xingyizhou/CenterNet/blob/master/readme/MODEL_ZOO.md) to ctdet_coco_dla_2x.onnx
        ```python
        from lib.opts import opts
        from lib.models.model import create_model, load_model
        from types import MethodType
        import torch.onnx as onnx
        import torch
        from torch.onnx import OperatorExportTypes
        from collections import OrderedDict
        import os
        os.environ.setdefault('CUDA_VISIBLE_DEVICES','3')
        ## copy the forward function of the model in lib/Models/networks/pose_dla_dcn.py.
        ## onnx is not support dict return value 
        def forward(self, x):
            x = self.base(x)
            x = self.dla_up(x)
            y = []
            for i in range(self.last_level - self.first_level):
                y.append(x[i].clone())
            self.ida_up(y, 0, len(y))
            ret = []                      ## change dict to list
            for head in self.heads:
                ret.append(self.__getattr__(head)(y[-1]))
            return ret
        
        opt = opts().init()
        print(opt)
        opt.arch = 'dla_34'
        opt.heads = OrderedDict([('hm',80),('reg',2),('wh',2)])
        model = create_model(opt.arch, opt.heads, opt.head_conv)
        model.forward = MethodType(forward,model)
        load_model(model, 'ctdet_coco_dla_2x.pth')
        model.eval()
        model.cuda()
        input = torch.zeros([1,3,512,512]).cuda()
        onnx.export(model,input,"ctdet_coco_dla_2x.onnx",verbose = True,
                   operator_export_type = OperatorExportTypes.ONNX )
        ```
    *   If you get (ValueError: Auto nesting doesn't know how to process an input object of type int. Accepted types: Tensors, or lists/tuples of them)
        You need to change (def _iter_filter) in torch.autograd.function.
        ```python
           def _iter_filter(....):
               ....
               if condition(obj):
                    yield obj
               elif isinstance(obj,int):  ## int to tensor
                    yield torch.tensor(obj)
               ....
   
        ```
4. onnx-tensorrt DCNv2 plugin
    * Related code
        * onnx-tensorrt/builtin_op_importers.cpp
        * onnx-tensorrt/builtin_plugins.cpp
        * onnx-tensorrt/DCNv2.hpp
        * onnx-tensorrt/DCNv2.cpp
        * onnx-tensorrt/dcn_v2_im2col_cuda.cu
        * onnx-tensorrt/dcn_v2_im2col_cuda.h
    * Not only support centernet. If you want to convert other model to tensorrt engine, please refer to src/ctdetNet.cpp or contact me.