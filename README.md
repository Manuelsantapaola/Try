public class Main {

    public static void main(String[] args) {

        Board board = new Board();

        Piece p1 = new Piece(1);   // 0001
        Piece p2 = new Piece(5);   // 0101
        Piece p3 = new Piece(9);   // 1001
        Piece p4 = new Piece(13);  // 1101

        board.placePiece(0, 0, p1);
        board.placePiece(0, 1, p2);
        board.placePiece(0, 2, p3);
        board.placePiece(0, 3, p4);

        board.printBoard();

        boolean win = WinChecker.hasQuarto(board);

        System.out.println();
        System.out.println("Quarto? " + win);
    }
}
