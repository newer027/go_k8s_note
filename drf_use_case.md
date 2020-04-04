# drf的场景分析

- 创建一个 filters.FilterSet，并在 viewsets.GenericViewSet 使用这个 filters.FilterSet。
```python
class OrderStartFilter(filters.FilterSet):
    min_create = filters.DateTimeFilter(field_name="create_time", lookup_expr='gte')
    max_create = filters.DateTimeFilter(field_name="create_time", lookup_expr='lte')
    min_status = filters.NumberFilter(field_name="status", lookup_expr='gte')
    
    class Meta:
        model = OrderStart
        fields = ['item_id', 'vendor', 'invoice_org', 'min_create', 'max_create', 'status', 'palate', 'serial_id', 'min_status', 'invoice_status']


class OrderStartViewSet(viewsets.GenericViewSet, mixins.ListModelMixin):
    permission_classes = (IsAuthenticated,)
    serializer_class = OrderStartSerializer
    queryset = OrderStart.objects.filter(valid=True)
    filterset_class = OrderStartFilter
    
    def get_queryset(self):
        vendor_qs = Vendor.objects.filter(user_settings__usernamename=str(self.request.user)).values('vendor')
        queryset = OrderStart.objects.filter(vendor__in=vendor_qs, valid=True)
        return queryset

    def create(self, request):
        if OrderStart.objects.filter(serial_id=request.data.get("serial_id")):
            OrderStart.objects.filter(serial_id=request.data.get("serial_id")).update(valid=False)
        if not Vendor.objects.filter(vendor=request.data.get("vendor")):
            Vendor.objects.create(vendor=request.data.get("vendor"))
        serializer = OrderStartSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

- 不调用drf，直接读取 request 信息，并用 JsonResponse 返回 json 对象。
```python
def user_tree(request):
    item_id = jwt.decode(request.META['HTTP_AUTHORIZATION'][7:], None, None)['user_id']
    user_id = User.objects.filter(id=item_id)[0].username
    if str(user_id) == 'ubuntu':
        userTree = [8, 7, 10, 9, 14, 1, 5, 3, 11, 4, 13, 6, 12]
    else:
        userTree = [2]
    return JsonResponse(userTree, safe=False)
```

- 通过 drf 中 APIView 方法，响应用户发起的 Get 请求。
```python
class LastestStatusDistance(APIView):
    def get(self, request, format=None):
        statusDistance = StatusDistance.objects.first()
        serializer = StatusDistanceSerializer(statusDistance)
        return Response(serializer.data)
```





