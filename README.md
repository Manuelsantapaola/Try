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
public static boolean hasQuarto(Board board) {

    // righe
    for (int row = 0; row < 4; row++) {

        if (isWinningLine(
                board.getPiece(row, 0),
                board.getPiece(row, 1),
                board.getPiece(row, 2),
                board.getPiece(row, 3))) {

            return true;
        }
    }

    // colonne
    for (int col = 0; col < 4; col++) {

        if (isWinningLine(
                board.getPiece(0, col),
                board.getPiece(1, col),
                board.getPiece(2, col),
                board.getPiece(3, col))) {

            return true;
        }
    }

    // diagonale principale
    if (isWinningLine(
            board.getPiece(0, 0),
            board.getPiece(1, 1),
            board.getPiece(2, 2),
            board.getPiece(3, 3))) {

        return true;
    }

    // diagonale secondaria
    return isWinningLine(
            board.getPiece(0, 3),
            board.getPiece(1, 2),
            board.getPiece(2, 1),
            board.getPiece(3, 0));
}
