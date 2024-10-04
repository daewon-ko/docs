# Mockito 관련

### 배경

쇼핑몰 토이프로젝트를 진행중에 있다. Controller Layer쪽 테스트 코드는 Mockito를 이용하여 검증 중에 있는데, 단순하게 아래와 같이 작성했는데 Assertions부에서 테스트가 지속해서 깨졌다. 원인은 아래와 같이 작성하면 data를 MockitoResponse객체가 응답해오지 못하다는 것이었다. 이유가 뭘까? 



~~~~java
    @DisplayName("페이지 사이즈 및 크기와 주문상태를 조건으로 사용자의 주문목록을 조회할 수 있다.")
    @Test
    void findPagedOrderProductsByUserIdAndStatus() throws Exception {
        //given

        // 신규생성된 주문을 1페이지부터 10개 가져온다.
        OrderSearchCondition orderSearchCondition = OrderSearchCondition.builder().orderStatus(OrderStatus.NEW).build();
        OrderProductResponseDto response1 = createOrderProductResponseDto(1L, 1L, "test", 99);
        OrderProductResponseDto response2 = createOrderProductResponseDto(2L, 2L, "test", 99);
        OrderProductResponseDto response3 = createOrderProductResponseDto(2L, 3L, "test", 100);
        PageRequest pageRequest = PageRequest.of(1, 3);


        Slice<OrderProductResponseDto> mockOrderList = new SliceImpl<>(List.of(response1, response2, response3));



        Mockito.when(orderService.getOrderList(1L, orderSearchCondition, pageRequest))
                        .thenReturn(mockOrderList);


//        Mockito.when(orderService.getOrderList(ArgumentMatchers.eq(1L),
//                        argThat(condition -> condition.getOrderStatus() == OrderStatus.NEW),
//                        argThat(pageable -> pageable.getPageNumber() ==1 && pageable.getPageSize() ==10)))
//                        .thenReturn(mockOrderList);


        //when
        mockMvc.perform(get("/api/order/{userId}/list", 1L)
                        .param("orderStatus", OrderStatus.NEW.name())
                        .param("page", "1")
                        .param("size", "10")
                        .contentType(MediaType.APPLICATION_JSON)
                )
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.data.content[0].orderId").value(1))
                .andExpect(jsonPath("$.data.content[0].quantity").value(99))
                .andExpect(jsonPath("$.data.content[1].orderId").value(2))
                .andExpect(jsonPath("$.data.content[1].quantity").value(99))
                .andExpect(jsonPath("$.data.content[2].orderId").value(2))
                .andExpect(jsonPath("$.data.content[2].quantity").value(100));

        //then

    }
~~~~

상단의 코드는 실패하는 테스트 코드이다. 주석처리가 된 부분으로 Mockito절을 변경하면 통과한다. 어떤 차이가 있던 걸까?